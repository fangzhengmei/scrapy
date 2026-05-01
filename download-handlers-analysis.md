# Scrapy 下载处理器多协议分发机制分析报告

## 1. 概述

Scrapy 的下载处理器（Download Handlers）是一套可插拔的组件系统，负责根据请求 URL 的协议（scheme）将请求路由到对应的处理器执行下载任务。本文档深入分析这套多协议分发机制的设计原理、实现细节以及 HTTP/2 支持的集成方式。

---

## 2. 核心分发机制

### 2.1 架构概览

```
                    ┌─────────────────────────────────────┐
                    │           Downloader                 │
                    │  (scrapy/core/downloader/__init__.py)│
                    └───────────────────┬─────────────────┘
                                        │
                                        ▼
                    ┌─────────────────────────────────────┐
                    │        DownloadHandlers             │
                    │  (scrapy/core/downloader/handlers/  │
                    │           __init__.py)               │
                    └───────────────────┬─────────────────┘
                                        │
            ┌───────────┬───────────────┼───────────────┬───────────┐
            │           │               │               │           │
            ▼           ▼               ▼               ▼           ▼
    ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌───────────┐
    │  HTTP11   │ │    H2     │ │   FTP     │ │   S3      │ │DataURI/   │
    │  Handler  │ │  Handler  │ │  Handler  │ │  Handler  │ │File       │
    └───────────┘ └───────────┘ └───────────┘ └───────────┘ └───────────┘
```

### 2.2 配置机制

Scrapy 通过两层配置来管理下载处理器：

#### 2.2.1 默认配置 (`DOWNLOAD_HANDLERS_BASE`)

在 `scrapy/settings/default_settings.py` 中定义了默认的协议映射：

```python
DOWNLOAD_HANDLERS_BASE = {
    "data": "scrapy.core.downloader.handlers.datauri.DataURIDownloadHandler",
    "file": "scrapy.core.downloader.handlers.file.FileDownloadHandler",
    "http": "scrapy.core.downloader.handlers.http11.HTTP11DownloadHandler",
    "https": "scrapy.core.downloader.handlers.http11.HTTP11DownloadHandler",
    "s3": "scrapy.core.downloader.handlers.s3.S3DownloadHandler",
    "ftp": "scrapy.core.downloader.handlers.ftp.FTPDownloadHandler",
}
```

| 协议 | 处理器类 | 说明 |
|------|----------|------|
| `data` | DataURIDownloadHandler | 处理 data: URI 协议 |
| `file` | FileDownloadHandler | 处理本地文件协议 |
| `http` | HTTP11DownloadHandler | 处理 HTTP/1.1 协议 |
| `https` | HTTP11DownloadHandler | 处理 HTTPS (HTTP/1.1) |
| `s3` | S3DownloadHandler | 处理 AWS S3 协议 |
| `ftp` | FTPDownloadHandler | 处理 FTP 协议 |

#### 2.2.2 用户自定义配置 (`DOWNLOAD_HANDLERS`)

用户可以通过 `DOWNLOAD_HANDLERS` 设置来覆盖或扩展默认配置：

```python
DOWNLOAD_HANDLERS = {
    # 禁用 FTP 支持
    "ftp": None,
    # 替换默认 HTTP 处理器
    "http": "my.custom.HttpHandler",
    # 添加新协议支持
    "sftp": "my.custom.SftpHandler",
}
```

---

## 3. 继承关系与职责定位（修正版）

### 3.1 继承层次结构

**关键修正**：`H2DownloadHandler` 直接继承 `BaseDownloadHandler`，**不是** `BaseHttpDownloadHandler`！

```
┌─────────────────────────────────────────────────────────────────┐
│                   BaseDownloadHandler (ABC)                      │
│  scrapy/core/downloader/handlers/base.py                        │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 属性:                                                     │   │
│  │   - lazy: bool = False (默认非惰性)                      │   │
│  │   - crawler: Crawler                                     │   │
│  │                                                          │   │
│  │ 方法:                                                     │   │
│  │   - __init__(crawler)                                    │   │
│  │   - from_crawler(cls, crawler) → Self                   │   │
│  │   - download_request(request) → Response (抽象方法)      │   │
│  │   - close() → None                                       │   │
│  └─────────────────────────────────────────────────────────┘   │
└───────────────────────────┬─────────────────────────────────────┘
                            │
            ┌───────────────┼───────────────┬───────────────────┐
            │               │               │                   │
            ▼               ▼               ▼                   ▼
┌─────────────────────┐ ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│ BaseHttpDownloader  │ │  H2Downloader   │ │ FTPDownloader   │ │ S3Downloader    │
│   (中间基类)         │ │                 │ │                 │ │                 │
│ scrapy/utils/        │ │  lazy = True    │ │  lazy = False   │ │  lazy = True    │
│ _download_handlers.py│ │                 │ │                 │ │                 │
└──────────┬──────────┘ └─────────────────┘ └─────────────────┘ └─────────────────┘
           │
           ▼
┌─────────────────────┐
│ HTTP11Downloader    │
│                     │
│  lazy = False       │
└─────────────────────┘

其他直接继承 BaseDownloadHandler 的处理器：
┌─────────────────────┐ ┌─────────────────────┐
│ DataURIDownloader   │ │ FileDownloader      │
│                     │ │                     │
│  lazy = False       │ │  lazy = False       │
└─────────────────────┘ └─────────────────────┘
```

### 3.2 各基类职责分析

#### 3.2.1 `BaseDownloadHandler` - 最底层抽象基类

**文件位置**：`scrapy/core/downloader/handlers/base.py`

```python
class BaseDownloadHandler(ABC):
    """Optional base class for download handlers."""

    lazy: bool = False

    def __init__(self, crawler: Crawler):
        self.crawler = crawler

    @classmethod
    def from_crawler(cls, crawler: Crawler) -> Self:
        return cls(crawler)

    @abstractmethod
    async def download_request(self, request: Request) -> Response:
        raise NotImplementedError

    async def close(self) -> None:
        pass
```

**职责**：
- 定义处理器的核心接口（Protocol）
- 提供 `crawler` 属性
- 提供默认的 `lazy = False`（非惰性加载）
- 提供默认的 `close()` 方法（空实现）

**所有处理器都必须实现的接口**：
```python
class DownloadHandlerProtocol(Protocol):
    lazy: bool
    async def download_request(self, request: Request) -> Response: ...
    async def close(self) -> None: ...
```

#### 3.2.2 `BaseHttpDownloadHandler` - HTTP 特定中间基类

**文件位置**：`scrapy/utils/_download_handlers.py`

```python
class BaseHttpDownloadHandler(BaseDownloadHandler, ABC):
    """Base class for built-in HTTP download handlers."""

    def __init__(self, crawler: Crawler):
        super().__init__(crawler)
        self._default_maxsize: int = crawler.settings.getint("DOWNLOAD_MAXSIZE")
        self._default_warnsize: int = crawler.settings.getint("DOWNLOAD_WARNSIZE")
        self._fail_on_dataloss: bool = crawler.settings.getbool(
            "DOWNLOAD_FAIL_ON_DATALOSS"
        )
        self._tls_verbose_logging: bool = crawler.settings.getbool(
            "DOWNLOADER_CLIENT_TLS_VERBOSE_LOGGING"
        )
        self._fail_on_dataloss_warned: bool = False
```

**职责**：
- 为 HTTP 相关处理器提供共享配置
- 从 settings 加载 HTTP 特定配置：
  - `_default_maxsize` - 最大下载大小
  - `_default_warnsize` - 警告大小
  - `_fail_on_dataloss` - 数据丢失时是否失败
  - `_tls_verbose_logging` - TLS 详细日志
- 标记为抽象类（ABC），不能直接实例化

### 3.3 各处理器继承关系汇总表

| 处理器 | 直接父类 | 间接父类 | lazy 值 | 使用 HTTP 特定配置 |
|--------|----------|----------|---------|-------------------|
| `HTTP11DownloadHandler` | `BaseHttpDownloadHandler` | `BaseDownloadHandler` | `False` | ✅ 是 |
| `H2DownloadHandler` | `BaseDownloadHandler` | 无 | `True` | ❌ 否（独立实现） |
| `FTPDownloadHandler` | `BaseDownloadHandler` | 无 | `False` | ❌ 否 |
| `S3DownloadHandler` | `BaseDownloadHandler` | 无 | `True` | ❌ 否 |
| `DataURIDownloadHandler` | `BaseDownloadHandler` | 无 | `False` | ❌ 否 |
| `FileDownloadHandler` | `BaseDownloadHandler` | 无 | `False` | ❌ 否 |

### 3.4 关键修正：H2DownloadHandler 的特殊设计

**之前的错误描述**：`H2DownloadHandler` 继承 `BaseHttpDownloadHandler`

**正确事实**：`H2DownloadHandler` 直接继承 `BaseDownloadHandler`，**不继承** `BaseHttpDownloadHandler`

#### 3.4.1 代码证据

```python
# scrapy/core/downloader/handlers/http2.py:31
class H2DownloadHandler(BaseDownloadHandler):  # 注意：是 BaseDownloadHandler，不是 BaseHttpDownloadHandler
    lazy = True

    def __init__(self, crawler: Crawler):
        if not crawler.settings.getbool("TWISTED_REACTOR_ENABLED"):
            raise NotConfigured(f"{type(self).__name__} requires a Twisted reactor.")
        super().__init__(crawler)  # 只调用 BaseDownloadHandler.__init__
        self._crawler = crawler

        from twisted.internet import reactor

        self._pool = H2ConnectionPool(reactor, crawler.settings)
        self._context_factory = _load_context_factory_from_settings(crawler)
        self._bind_address = crawler.settings.get("DOWNLOAD_BIND_ADDRESS")
```

对比 `HTTP11DownloadHandler`：

```python
# scrapy/core/downloader/handlers/http11.py:83
class HTTP11DownloadHandler(BaseHttpDownloadHandler):  # 继承 BaseHttpDownloadHandler
    def __init__(self, crawler: Crawler):
        if not crawler.settings.getbool("TWISTED_REACTOR_ENABLED"):
            raise NotConfigured(f"{type(self).__name__} requires a Twisted reactor.")
        super().__init__(crawler)  # 调用 BaseHttpDownloadHandler.__init__
        # 可以使用 self._default_maxsize, self._default_warnsize 等
```

#### 3.4.2 H2DownloadHandler 为何不继承 BaseHttpDownloadHandler？

**设计原因分析**：

1. **HTTP/2 的 maxsize 处理机制不同**：
   - HTTP/1.1：在 `ScrapyAgent` 和 `_ResponseReader` 中处理，使用继承的 `_default_maxsize`
   - HTTP/2：在 `Stream` 类中独立处理，从请求 meta 或 spider 属性获取

   ```python
   # scrapy/core/http2/stream.py:97-120
   class Stream:
       def __init__(self, stream_id: int, request: Request, protocol: H2ClientProtocol, ...):
           # 从请求 meta 获取，不是从 handler 继承
           self._download_maxsize = self._request.meta.get(
               "download_maxsize", download_maxsize
           )
           self._download_warnsize = self._request.meta.get(
               "download_warnsize", download_warnsize
           )
   ```

2. **HTTP/2 不支持所有 HTTP/1.1 的特性**：
   - 不支持 `bytes_received` 和 `headers_received` 信号
   - 不支持 `stop_download` 机制（`BaseHttpDownloadHandler` 相关的 `check_stop_download`）

3. **HTTP/2 是实验性功能**：
   - 文档标记为 "experimental"
   - 单独的模块结构（`scrapy/core/http2/`）
   - 需要手动配置启用

#### 3.4.3 H2DownloadHandler 的职责定位

| 职责 | 实现方式 |
|------|----------|
| **协议分发兼容** | 实现 `DownloadHandlerProtocol`，可被 `DownloadHandlers` 路由 |
| **连接管理** | 拥有独立的 `H2ConnectionPool`，支持多路复用 |
| **TLS 配置** | 通过 `_load_context_factory_from_settings` 加载，强制 ALPN 协商 `h2` |
| **超时处理** | 在 `ScrapyH2Agent` 中独立实现（`download_request` 中的 `callLater`） |
| **大小限制** | 在 `Stream` 类中独立实现，不从 `BaseHttpDownloadHandler` 继承 |

---

## 4. 协议路由机制（复核版）

### 4.1 核心实现

`DownloadHandlers` 类位于 `scrapy/core/downloader/handlers/__init__.py`，是整个分发机制的核心。

#### 4.1.1 数据结构

```python
class DownloadHandlers:
    def __init__(self, crawler: Crawler):
        self._crawler: Crawler = crawler
        
        # 协议到处理器类路径的映射（加载自配置）
        self._schemes: dict[str, str | Callable[..., Any]] = {}
        
        # 已实例化的处理器缓存（协议 -> 处理器实例）
        self._handlers: dict[str, DownloadHandlerProtocol] = {}
        
        # 配置失败的协议及其原因
        self._notconfigured: dict[str, str] = {}
        
        # 旧风格处理器标记（返回 Deferred 而非协程）
        self._old_style_handlers: set[str] = set()
```

#### 4.1.2 初始化流程

```python
def __init__(self, crawler: Crawler):
    # 1. 合并配置：DOWNLOAD_HANDLERS + DOWNLOAD_HANDLERS_BASE
    handlers: dict[str, str | Callable[..., Any]] = without_none_values(
        cast(
            "dict[str, str | Callable[..., Any]]",
            crawler.settings.getwithbase("DOWNLOAD_HANDLERS"),
        )
    )
    
    # 2. 遍历所有协议，注册到 _schemes
    for scheme, clspath in handlers.items():
        self._schemes[scheme] = clspath
        # 3. 尝试预加载（非惰性处理器会被实例化）
        self._load_handler(scheme, skip_lazy=True)
    
    # 4. 注册关闭信号
    crawler.signals.connect(self._close, signals.engine_stopped)
```

#### 4.1.3 请求路由流程

```python
async def download_request_async(self, request: Request) -> Response:
    # 步骤 1：解析 URL 获取协议
    scheme = urlparse_cached(request).scheme
    
    # 步骤 2：获取对应处理器（支持惰性加载）
    handler = self._get_handler(scheme)
    
    # 步骤 3：检查处理器是否可用
    if not handler:
        raise NotSupported(
            f"Unsupported URL scheme '{scheme}': {self._notconfigured[scheme]}"
        )
    
    # 步骤 4：调用处理器执行下载
    assert self._crawler.spider
    
    # 兼容旧风格处理器（返回 Deferred）
    if scheme in self._old_style_handlers:
        return await maybe_deferred_to_future(
            cast(
                "Deferred[Response]",
                handler.download_request(request, self._crawler.spider),
            )
        )
    
    # 新风格：协程
    return await handler.download_request(request)
```

### 4.2 路由流程图

```
请求 URL: "https://example.com/page"
           │
           ▼
┌─────────────────────────────────────────────────────────────┐
│  urlparse_cached(request).scheme                             │
│  结果: "https"                                                │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  _get_handler("https")                                       │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ 1. 检查 self._handlers 中是否已有缓存               │    │
│  │    if "https" in self._handlers: return it         │    │
│  │                                                      │    │
│  │ 2. 检查是否配置失败                                   │    │
│  │    if "https" in self._notconfigured: return None  │    │
│  │                                                      │    │
│  │ 3. 检查是否支持该协议                                 │    │
│  │    if "https" not in self._schemes:                 │    │
│  │        self._notconfigured["https"] = "no handler"  │    │
│  │        return None                                   │    │
│  │                                                      │    │
│  │ 4. 惰性加载处理器                                     │    │
│  │    return self._load_handler("https")               │    │
│  └─────────────────────────────────────────────────────┘    │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  处理器实例: HTTP11DownloadHandler 或 H2DownloadHandler      │
│  (取决于用户配置的 DOWNLOAD_HANDLERS)                        │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  handler.download_request(request)                           │
│  执行实际下载                                                 │
└─────────────────────────────────────────────────────────────┘
```

### 4.3 配置合并规则

`settings.getwithbase("DOWNLOAD_HANDLERS")` 的合并逻辑：

```python
# 伪代码逻辑
def getwithbase(key):
    base_value = getattr(self, key + "_BASE", {})
    user_value = getattr(self, key, {})
    
    # 合并：用户配置覆盖默认配置
    result = base_value.copy()
    result.update(user_value)
    
    # 过滤掉值为 None 的项（表示禁用）
    return {k: v for k, v in result.items() if v is not None}
```

**示例**：

```python
# 默认配置 (DOWNLOAD_HANDLERS_BASE)
{
    "http": "HTTP11DownloadHandler",
    "https": "HTTP11DownloadHandler",
    "ftp": "FTPDownloadHandler",
    ...
}

# 用户配置 (DOWNLOAD_HANDLERS)
{
    "https": "H2DownloadHandler",  # 覆盖默认
    "ftp": None,                     # 禁用
    "sftp": "MySftpHandler"          # 新增
}

# 合并结果
{
    "http": "HTTP11DownloadHandler",   # 保留默认
    "https": "H2DownloadHandler",      # 用户覆盖
    # "ftp" 被移除（值为 None）
    "sftp": "MySftpHandler",           # 用户新增
    ...
}
```

---

## 5. 惰性加载机制（复核版）

### 5.1 设计目的

惰性加载（Lazy Loading）的设计目的：

1. **减少启动时间**：只在需要时才实例化处理器
2. **节省资源**：对于不使用的协议（如 S3），不会加载其依赖
3. **错误隔离**：某个处理器初始化失败不影响其他处理器

### 5.2 实现机制

#### 5.2.1 `lazy` 属性

每个处理器通过 `lazy` 类属性声明是否惰性加载：

```python
# 惰性加载示例
class H2DownloadHandler(BaseDownloadHandler):
    lazy = True  # 启用惰性加载

class S3DownloadHandler(BaseDownloadHandler):
    lazy = True  # 启用惰性加载

# 非惰性加载示例
class HTTP11DownloadHandler(BaseHttpDownloadHandler):
    # 继承 BaseHttpDownloadHandler，默认 lazy = False
    pass

class BaseDownloadHandler(ABC):
    lazy: bool = False  # 默认非惰性
```

#### 5.2.2 `_load_handler` 方法核心逻辑

```python
def _load_handler(
    self, scheme: str, skip_lazy: bool = False
) -> DownloadHandlerProtocol | None:
    path = self._schemes[scheme]
    
    try:
        # 1. 加载处理器类
        dhcls: type[DownloadHandlerProtocol] = load_object(path)
        
        # 2. 检查是否跳过惰性加载（初始化阶段调用时 skip_lazy=True）
        if skip_lazy:
            # 2.1 检查是否定义了 lazy 属性（兼容性警告）
            if not hasattr(dhcls, "lazy"):
                warnings.warn(
                    f"{global_object_name(dhcls)} doesn't define a 'lazy' attribute."
                    f" This is deprecated, please add 'lazy = True'...",
                    category=ScrapyDeprecationWarning,
                    stacklevel=1,
                )
            
            # 2.2 如果是惰性处理器，返回 None（不实例化）
            # 注意：getattr(dhcls, "lazy", True) 意味着未定义时默认 True
            if getattr(dhcls, "lazy", True):
                return None
        
        # 3. 实例化处理器
        dh = build_from_crawler(dhcls, self._crawler)
    
    except NotConfigured as ex:
        self._notconfigured[scheme] = str(ex)
        return None
    except Exception as ex:
        logger.error('Loading "%(clspath)s" for scheme "%(scheme)s"', ...)
        self._notconfigured[scheme] = str(ex)
        return None
    
    # 4. 缓存实例
    self._handlers[scheme] = dh
    
    # 5. 检查是否为旧风格处理器
    if not inspect.iscoroutinefunction(dh.download_request):
        warnings.warn(...)
        self._old_style_handlers.add(scheme)
    
    return dh
```

### 5.3 加载时序图

```
Scrapy 启动阶段
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│  DownloadHandlers.__init__(crawler)                          │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ for scheme, clspath in handlers.items():            │    │
│  │     self._schemes[scheme] = clspath                 │    │
│  │     self._load_handler(scheme, skip_lazy=True)      │    │
│  │                                                       │    │
│  │  在 _load_handler 中 (skip_lazy=True):               │    │
│  │  - 如果 handler.lazy == True: return None (不实例化)│    │
│  │  - 如果 handler.lazy == False: 实例化并缓存         │    │
│  └─────────────────────────────────────────────────────┘    │
└───────────────────────────┬─────────────────────────────────┘
                            │
    ┌───────────────────────┴───────────────────────┐
    │                                               │
    ▼                                               ▼
非惰性处理器已实例化                        惰性处理器未实例化
(HTTP11, FTP, DataURI, File)               (H2, S3)
    │                                               │
    │                                               ▼
    │                                    第一个该协议请求到达时
    │                                               │
    │                                               ▼
    │                              ┌─────────────────────────────────┐
    │                              │  _get_handler(scheme)          │
    │                              │  → _load_handler(scheme,       │
    │                              │         skip_lazy=False)      │
    │                              │  → 实例化并缓存                │
    │                              └─────────────────────────────────┘
    │                                               │
    └───────────────────────┬───────────────────────┘
                            │
                            ▼
                    处理器准备就绪
```

### 5.4 惰性加载处理器列表

| 处理器 | lazy 值 | 原因 |
|--------|----------|------|
| `H2DownloadHandler` | `True` | 实验性功能，默认不启用 |
| `S3DownloadHandler` | `True` | 需要 `botocore` 依赖，可能未安装 |
| `HTTP11DownloadHandler` | `False` | 默认 HTTP 处理器，总是需要 |
| `FTPDownloadHandler` | `False` | 默认协议处理器 |
| `DataURIDownloadHandler` | `False` | 轻量级，无依赖 |
| `FileDownloadHandler` | `False` | 轻量级，无依赖 |

### 5.5 S3 处理器的特殊惰性行为

`S3DownloadHandler` 不仅在类级别惰性加载，实例化时还会检查依赖：

```python
class S3DownloadHandler(BaseDownloadHandler):
    lazy = True

    def __init__(self, crawler: Crawler):
        # 检查 botocore 是否可用
        if not is_botocore_available():
            raise NotConfigured("missing botocore library")
        
        super().__init__(crawler)
        # ... AWS 签名器初始化
        
        # 动态加载配置的 HTTPS 处理器
        _http_handler = build_from_crawler(
            load_object(crawler.settings.getwithbase("DOWNLOAD_HANDLERS")["https"]),
            crawler,
        )
        self._download_http = _http_handler.download_request
```

**这意味着**：
1. 类级别：`lazy = True` → Scrapy 启动时不实例化
2. 实例级别：如果 `botocore` 未安装 → 抛出 `NotConfigured` → 被 `_notconfigured` 记录
3. 动态依赖：运行时加载当前配置的 HTTPS 处理器

---

## 6. HTTP/2 支持集成分析（修正版）

### 6.1 概述

HTTP/2 支持在 Scrapy 中是**实验性功能**，具有以下特点：

1. **默认未启用**：需要手动配置
2. **独立实现**：有自己的连接池、协议、流管理
3. **仅支持 HTTPS**：不支持明文 HTTP/2 (h2c)
4. **惰性加载**：`lazy = True`

### 6.2 配置方式

```python
# settings.py
DOWNLOAD_HANDLERS = {
    "https": "scrapy.core.downloader.handlers.http2.H2DownloadHandler",
}
```

**重要**：HTTP/2 处理器只支持 `https` 协议，不支持 `http` 协议。

### 6.3 模块架构

```
scrapy/core/downloader/handlers/http2.py
    │
    ├── H2DownloadHandler (继承 BaseDownloadHandler)
    │   ├── lazy = True
    │   ├── __init__(crawler)
    │   │   ├── 检查 TWISTED_REACTOR_ENABLED
    │   │   ├── 创建 H2ConnectionPool
    │   │   └── 加载 TLS 上下文工厂
    │   ├── download_request(request) → Response
    │   └── close() → None
    │
    └── ScrapyH2Agent (封装类)
        ├── _get_agent(request, timeout) → H2Agent
        └── download_request(request, spider) → Deferred[Response]

scrapy/core/http2/
    │
    ├── agent.py
    │   ├── H2ConnectionPool (连接池)
    │   │   ├── _connections: dict[ConnectionKeyT, H2ClientProtocol]
    │   │   ├── _pending_requests: dict[ConnectionKeyT, deque[Deferred]]
    │   │   ├── get_connection(key, uri, endpoint)
    │   │   ├── _new_connection(key, uri, endpoint)
    │   │   └── close_connections()
    │   │
    │   ├── H2Agent (核心 Agent)
    │   │   ├── endpoint_factory (用于创建端点)
    │   │   ├── get_endpoint(uri) → HostnameEndpoint
    │   │   ├── get_key(uri) → ConnectionKeyT
    │   │   └── request(request, spider) → Deferred[Response]
    │   │
    │   └── ScrapyProxyH2Agent (HTTP 代理支持)
    │       └── 重写 get_endpoint 和 get_key
    │
    ├── protocol.py
    │   ├── H2ClientProtocol (协议实现)
    │   │   ├── conn: H2Connection (hyper-h2)
    │   │   ├── streams: dict[int, Stream]
    │   │   ├── _pending_request_stream_pool: deque[Stream]
    │   │   ├── request(request, spider) → Deferred[Response]
    │   │   ├── dataReceived(data)
    │   │   └── _handle_events(events)
    │   │
    │   └── H2ClientFactory (协议工厂)
    │       └── acceptableProtocols() → [b"h2"]
    │
    └── stream.py
        └── Stream (单个 HTTP/2 流)
            ├── stream_id: int
            ├── _download_maxsize: int
            ├── _download_warnsize: int
            ├── initiate_request()
            ├── send_data()
            ├── receive_headers(headers)
            ├── receive_data(data, flow_controlled_length)
            └── close(reason, errors, from_protocol)
```

### 6.4 H2DownloadHandler 详细实现

#### 6.4.1 初始化

```python
class H2DownloadHandler(BaseDownloadHandler):  # 注意：不是 BaseHttpDownloadHandler
    lazy = True

    def __init__(self, crawler: Crawler):
        # 1. 检查 Twisted reactor 是否启用
        if not crawler.settings.getbool("TWISTED_REACTOR_ENABLED"):
            raise NotConfigured(f"{type(self).__name__} requires a Twisted reactor.")
        
        # 2. 调用父类初始化（只设置 self.crawler）
        super().__init__(crawler)
        self._crawler = crawler

        from twisted.internet import reactor

        # 3. 创建 HTTP/2 连接池
        self._pool = H2ConnectionPool(reactor, crawler.settings)
        
        # 4. 加载 TLS 上下文工厂
        # 注意：这里会强制设置 ALPN 协议为 [b"h2"]
        self._context_factory = _load_context_factory_from_settings(crawler)
        
        # 5. 绑定地址
        self._bind_address = crawler.settings.get("DOWNLOAD_BIND_ADDRESS")
```

#### 6.4.2 下载请求处理

```python
async def download_request(self, request: Request) -> Response:
    # 创建 ScrapyH2Agent 实例
    agent = ScrapyH2Agent(
        context_factory=self._context_factory,
        pool=self._pool,
        bind_address=self._bind_address,
        crawler=self._crawler,
    )
    
    assert self._crawler.spider
    
    # 执行下载（使用 wrap_twisted_exceptions 包装异常）
    with wrap_twisted_exceptions():
        return await maybe_deferred_to_future(
            agent.download_request(request, self._crawler.spider)
        )
```

### 6.5 ALPN 协议协商

HTTP/2 强制使用 ALPN (Application Layer Protocol Negotiation) 来协商协议。

#### 6.5.1 上下文工厂

```python
# scrapy/core/downloader/contextfactory.py

class _AcceptableProtocolsContextFactory:
    """用于强制协商特定协议的上下文工厂包装器"""
    
    def __init__(self, context_factory, acceptable_protocols):
        self._context_factory = context_factory
        self._acceptable_protocols = acceptable_protocols

    def creatorForNetloc(self, hostname, port):
        # 获取原始 SSL 上下文
        context = self._context_factory.creatorForNetloc(hostname, port)
        
        # 强制设置 ALPN 协议
        context.set_alpn_protocols(self._acceptable_protocols)
        
        # 强制设置 NPN 协议（向后兼容）
        context.set_npn_protocols(self._acceptable_protocols)
        
        return context
```

#### 6.5.2 H2Agent 中的应用

```python
# scrapy/core/http2/agent.py

class H2Agent:
    def __init__(
        self,
        reactor: ReactorBase,
        pool: H2ConnectionPool,
        context_factory: BrowserLikePolicyForHTTPS = BrowserLikePolicyForHTTPS(),
        connect_timeout: float | None = None,
        bind_address: tuple[str, int] | None = None,
    ) -> None:
        self._reactor = reactor
        self._pool = pool
        
        # 关键：包装上下文工厂，强制协商 h2 协议
        self._context_factory = _AcceptableProtocolsContextFactory(
            context_factory, acceptable_protocols=[b"h2"]
        )
        
        self.endpoint_factory = _StandardEndpointFactory(
            self._reactor, self._context_factory, connect_timeout, bind_address
        )
```

#### 6.5.3 协议验证

如果服务器不支持 HTTP/2，连接会失败：

```python
# scrapy/core/http2/protocol.py

class H2ClientProtocol(Protocol, TimeoutMixin):
    # ...
    
    def handshakeCompleted(self) -> None:
        """TLS 握手完成后验证协商的协议"""
        assert self.transport is not None
        
        # 检查协商的协议是否为 h2
        if (
            self.transport.negotiatedProtocol is not None
            and self.transport.negotiatedProtocol != PROTOCOL_NAME  # PROTOCOL_NAME = b"h2"
        ):
            # 协议不匹配，关闭连接
            self._lose_connection_with_error(
                [InvalidNegotiatedProtocol(self.transport.negotiatedProtocol)]
            )
```

### 6.6 连接池管理

HTTP/2 的连接池与 HTTP/1.1 有本质区别：

| 特性 | HTTP/1.1 | HTTP/2 |
|------|----------|--------|
| 连接复用 | 每个请求一个连接（或有限持久连接） | 单连接多路复用 |
| 并发模型 | 依赖多个 TCP 连接 | 单连接多流 |
| 连接键 | 主机+端口 | 主机+端口+协议 |
| 待处理请求 | 无（立即创建新连接） | 有（等待连接建立） |

#### 6.6.1 H2ConnectionPool 实现

```python
class H2ConnectionPool:
    def __init__(self, reactor: ReactorBase, settings: Settings):
        self._reactor = reactor
        self.settings = settings
        
        # 活跃连接：key -> H2ClientProtocol
        # key = (scheme, host, port)
        self._connections: dict[ConnectionKeyT, H2ClientProtocol] = {}
        
        # 待处理请求：在连接建立前到达的请求
        self._pending_requests: dict[
            ConnectionKeyT, deque[Deferred[H2ClientProtocol]]
        ] = {}

    def get_connection(
        self, key: ConnectionKeyT, uri: URI, endpoint: HostnameEndpoint
    ) -> Deferred[H2ClientProtocol]:
        # 1. 检查是否有正在建立的连接
        if key in self._pending_requests:
            # 加入待处理队列
            d: Deferred[H2ClientProtocol] = Deferred()
            self._pending_requests[key].append(d)
            return d
        
        # 2. 检查是否已有可用连接
        conn = self._connections.get(key, None)
        if conn:
            return defer.succeed(conn)
        
        # 3. 建立新连接
        return self._new_connection(key, uri, endpoint)
```

### 6.7 多路复用与流管理

#### 6.7.1 Stream 类

每个 HTTP/2 请求对应一个 `Stream` 对象：

```python
class Stream:
    def __init__(
        self,
        stream_id: int,
        request: Request,
        protocol: H2ClientProtocol,
        download_maxsize: int = 0,
        download_warnsize: int = 0,
    ) -> None:
        self.stream_id = stream_id  # 奇数，客户端发起
        self._request = request
        self._protocol = protocol
        
        # 注意：这些值从参数传入，不是从 BaseHttpDownloadHandler 继承
        self._download_maxsize = self._request.meta.get(
            "download_maxsize", download_maxsize
        )
        self._download_warnsize = self._request.meta.get(
            "download_warnsize", download_warnsize
        )
        
        # 响应缓冲区
        self._response: dict[str, Any] = {
            "body": BytesIO(),
            "flow_controlled_size": 0,
            "headers": Headers(),
            "status": None,
        }
        
        # 响应 Deferred
        self._deferred_response: Deferred[Response] = Deferred(_cancel)
```

#### 6.7.2 并发流控制

```python
class H2ClientProtocol(Protocol, TimeoutMixin):
    # ...
    
    @property
    def allowed_max_concurrent_streams(self) -> int:
        """根据 SETTINGS 帧确定最大并发流数"""
        return min(
            self.conn.local_settings.max_concurrent_streams,
            self.conn.remote_settings.max_concurrent_streams,
        )

    def _send_pending_requests(self) -> None:
        """根据并发限制发送待处理请求"""
        while (
            self._pending_request_stream_pool
            and self.metadata["active_streams"] < self.allowed_max_concurrent_streams
            and self.h2_connected
        ):
            self.metadata["active_streams"] += 1
            stream = self._pending_request_stream_pool.popleft()
            stream.initiate_request()
            self._write_to_transport()
```

### 6.8 HTTP/2 与 HTTP/1.1 处理器对比

| 特性 | HTTP11DownloadHandler | H2DownloadHandler |
|------|----------------------|-------------------|
| **继承关系** | `BaseHttpDownloadHandler` | `BaseDownloadHandler` |
| **HTTP 特定配置** | 继承 `_default_maxsize` 等 | 不继承，独立实现 |
| **默认启用** | ✅ 是 | ❌ 否，需手动配置 |
| **惰性加载** | `lazy = False` | `lazy = True` |
| **连接池** | `HTTPConnectionPool` (Twisted) | `H2ConnectionPool` (自定义) |
| **多路复用** | ❌ 不支持 | ✅ 支持 (Stream) |
| **头部压缩** | ❌ 无 | ✅ HPACK |
| **信号支持** | ✅ `bytes_received`, `headers_received` | ❌ 不支持 |
| **代理支持** | ✅ 完整（HTTP + HTTPS 隧道） | ⚠️ 仅 HTTP 代理 |
| **依赖** | Twisted | Twisted + `h2` + `priority` |

### 6.9 已知限制

根据文档，HTTP/2 处理器存在以下限制：

1. **不支持明文 HTTP/2 (h2c)**：因为主流浏览器都不支持未加密的 HTTP/2
2. **不支持更大帧大小**：连接到发送大于默认 16384 字节帧的服务器会失败
3. **服务器推送被忽略**：虽然 HTTP/2 支持服务器推送，但 Scrapy 会忽略这些推送
4. **不支持部分信号**：`bytes_received` 和 `headers_received` 信号不可用
5. **HTTPS 隧道代理不支持**：无法通过 CONNECT 方法使用 HTTP/2 隧道

---

## 7. 跨模块协作分析（复核版）

### 7.1 整体调用链

```
请求进入
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│                    Downloader.fetch()                         │
│  scrapy/core/downloader/__init__.py:126                      │
│                                                              │
│  职责：                                                        │
│  - 管理下载槽位 (Slot) 和并发控制                              │
│  - 通过下载中间件链                                            │
│  - 触发信号 (request_reached_downloader 等)                   │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│            DownloaderMiddlewareManager.download_async()      │
│                                                              │
│  职责：                                                        │
│  - 执行下载中间件（请求处理、响应处理、异常处理）               │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│              Downloader._enqueue_request()                    │
│  scrapy/core/downloader/__init__.py:176                      │
│                                                              │
│  职责：                                                        │
│  - 获取或创建下载槽位 (Slot)                                   │
│  - 将请求入队                                                  │
│  - 触发队列处理                                                │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                Downloader._download()                         │
│  scrapy/core/downloader/__init__.py:221                      │
│                                                              │
│  职责：                                                        │
│  - 标记请求为传输中                                            │
│  - 调用处理器执行下载                                          │
│  - 触发信号 (response_downloaded, request_left_downloader)    │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│       DownloadHandlers.download_request_async()              │
│  scrapy/core/downloader/handlers/__init__.py:141            │
│                                                              │
│  职责：                                                        │
│  - 解析 URL 协议 (scheme)                                      │
│  - 路由到对应处理器（支持惰性加载）                             │
│  - 调用处理器的 download_request()                             │
└───────────────────────────┬─────────────────────────────────┘
                            │
            ┌───────────────┼───────────────┬───────────────┐
            │               │               │               │
            ▼               ▼               ▼               ▼
    ┌───────────┐   ┌───────────┐   ┌───────────┐   ┌───────────┐
    │ HTTP11    │   │   H2      │   │   FTP     │   │   S3      │
    │ Handler   │   │  Handler  │   │  Handler  │   │  Handler  │
    └───────────┘   └───────────┘   └───────────┘   └─────┬─────┘
                                                              │
                                                              ▼
                                                      ┌───────────┐
                                                      │ HTTP11    │
                                                      │  Handler  │
                                                      │ (委托)    │
                                                      └───────────┘
```

### 7.2 模块依赖关系图

```
scrapy.core.downloader.Downloader
    │
    ├── scrapy.core.downloader.handlers.DownloadHandlers (核心分发器)
    │       │
    │       ├── scrapy.core.downloader.handlers.http11.HTTP11DownloadHandler
    │       │       ├── scrapy.utils._download_handlers.BaseHttpDownloadHandler
    │       │       │       └── scrapy.core.downloader.handlers.base.BaseDownloadHandler
    │       │       ├── twisted.web.client.Agent
    │       │       ├── twisted.web.client.HTTPConnectionPool
    │       │       └── scrapy.core.downloader.contextfactory (TLS)
    │       │
    │       ├── scrapy.core.downloader.handlers.http2.H2DownloadHandler
    │       │       ├── scrapy.core.downloader.handlers.base.BaseDownloadHandler (直接)
    │       │       ├── scrapy.core.http2.agent.H2Agent
    │       │       │       ├── scrapy.core.http2.agent.H2ConnectionPool
    │       │       │       └── scrapy.core.http2.protocol.H2ClientProtocol
    │       │       │               ├── scrapy.core.http2.stream.Stream
    │       │       │               └── h2.connection.H2Connection (hyper-h2 库)
    │       │       └── scrapy.core.downloader.contextfactory._AcceptableProtocolsContextFactory
    │       │
    │       ├── scrapy.core.downloader.handlers.ftp.FTPDownloadHandler
    │       │       ├── scrapy.core.downloader.handlers.base.BaseDownloadHandler
    │       │       └── twisted.protocols.ftp.FTPClient
    │       │
    │       ├── scrapy.core.downloader.handlers.s3.S3DownloadHandler
    │       │       ├── scrapy.core.downloader.handlers.base.BaseDownloadHandler
    │       │       ├── botocore.auth (AWS 签名)
    │       │       └── [动态委托给配置的 HTTPS 处理器]
    │       │
    │       ├── scrapy.core.downloader.handlers.datauri.DataURIDownloadHandler
    │       │       ├── scrapy.core.downloader.handlers.base.BaseDownloadHandler
    │       │       └── w3lib.url.parse_data_uri
    │       │
    │       └── scrapy.core.downloader.handlers.file.FileDownloadHandler
    │               ├── scrapy.core.downloader.handlers.base.BaseDownloadHandler
    │               └── w3lib.url.file_uri_to_path
    │
    └── scrapy.core.downloader.middleware.DownloaderMiddlewareManager
```

### 7.3 共享组件详解

#### 7.3.1 `wrap_twisted_exceptions`

**位置**：`scrapy/utils/_download_handlers.py`

**职责**：将 Twisted 异常转换为 Scrapy 标准异常

```python
@contextmanager
def wrap_twisted_exceptions() -> Iterator[None]:
    try:
        yield
    except SchemeNotSupported as e:
        raise UnsupportedURLSchemeError(str(e)) from e
    except CancelledError as e:
        raise DownloadCancelledError(str(e)) from e
    except TxConnectionRefusedError as e:
        raise DownloadConnectionRefusedError(str(e)) from e
    except DNSLookupError as e:
        raise CannotResolveHostError(str(e)) from e
    except ResponseFailed as e:
        raise DownloadFailedError(str(e)) from e
    except TxTimeoutError as e:
        raise DownloadTimeoutError(str(e)) from e
```

**使用者**：
- `HTTP11DownloadHandler` ✅
- `H2DownloadHandler` ✅
- 其他处理器 ❌（不使用 Twisted 网络）

#### 7.3.2 `make_response`

**位置**：`scrapy/utils/_download_handlers.py`

**职责**：构建标准 Scrapy 响应对象

```python
def make_response(
    url: str,
    status: int,
    headers: Headers,
    body: bytes = b"",
    flags: list[str] | None = None,
    certificate: Certificate | None = None,
    ip_address: IPv4Address | IPv6Address | None = None,
    protocol: str | None = None,
    stop_download: StopDownload | None = None,
) -> Response:
    # 根据内容类型选择响应类
    respcls = responsetypes.from_args(headers=headers, url=url, body=body)
    
    # 构建响应
    response = respcls(
        url=url,
        status=status,
        headers=headers,
        body=body,
        flags=flags,
        certificate=certificate,
        ip_address=ip_address,
        protocol=protocol,
    )
    
    # 处理 stop_download
    if stop_download:
        response.flags.append("download_stopped")
        if stop_download.fail:
            stop_download.response = response
            raise stop_download
    
    return response
```

**使用者**：
- `HTTP11DownloadHandler` (通过 `ScrapyAgent`) ✅
- `H2DownloadHandler` (通过 `Stream`) ✅
- `FTPDownloadHandler` ❌（自己构建响应）
- `DataURIDownloadHandler` ❌（自己构建响应）
- `FileDownloadHandler` ❌（自己构建响应）

#### 7.3.3 `check_stop_download`

**位置**：`scrapy/utils/_download_handlers.py`

**职责**：检查信号处理器是否抛出 `StopDownload` 异常

```python
def check_stop_download(
    signal: object, crawler: Crawler, request: Request, **kwargs: Any
) -> StopDownload | None:
    signal_result = crawler.signals.send_catch_log(
        signal=signal,
        request=request,
        spider=crawler.spider,
        **kwargs,
    )
    for handler, result in signal_result:
        if isinstance(result, Failure) and isinstance(result.value, StopDownload):
            logger.debug(
                f"Download stopped for {request} from signal handler {handler.__qualname__}"
            )
            return result.value
    
    return None
```

**使用者**：
- `HTTP11DownloadHandler` ✅（支持 `headers_received` 和 `bytes_received` 信号）
- `H2DownloadHandler` ❌（不支持这些信号）

#### 7.3.4 `_load_context_factory_from_settings`

**位置**：`scrapy/core/downloader/contextfactory.py`

**职责**：从设置加载 TLS 上下文工厂

```python
def _load_context_factory_from_settings(crawler: Crawler) -> IPolicyForHTTPS:
    client_context_factory = crawler.settings.get(
        "DOWNLOADER_CLIENTCONTEXTFACTORY"
    )
    
    if client_context_factory == "SENTINEL":
        # 默认：使用 ScrapyClientContextFactory
        return ScrapyClientContextFactory(
            crawler.settings.get("DOWNLOADER_CLIENT_TLS_METHOD"),
            crawler.settings.get("DOWNLOADER_CLIENT_TLS_VERBOSE_LOGGING"),
            crawler.settings.get("DOWNLOADER_CLIENT_TLS_CIPHERS"),
        )
    else:
        # 自定义上下文工厂
        return build_from_crawler(
            load_object(client_context_factory), crawler
        )
```

**使用者**：
- `HTTP11DownloadHandler` ✅
- `H2DownloadHandler` ✅

### 7.4 配置加载流程

```
┌─────────────────────────────────────────────────────────────┐
│  用户 settings.py                                             │
│                                                              │
│  DOWNLOAD_HANDLERS = {                                        │
│      "https": "scrapy.core.downloader.handlers.http2.H2..." │
│  }                                                            │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  BaseSettings.getwithbase("DOWNLOAD_HANDLERS")              │
│                                                              │
│  逻辑：                                                       │
│  1. 获取 DOWNLOAD_HANDLERS_BASE（默认配置）                  │
│  2. 获取 DOWNLOAD_HANDLERS（用户配置）                       │
│  3. 合并：用户配置覆盖默认配置                                 │
│  4. 过滤：移除值为 None 的项（表示禁用）                       │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  DownloadHandlers.__init__()                                 │
│                                                              │
│  1. 调用 without_none_values() 确保没有 None 值              │
│  2. 遍历协议-处理器映射：                                      │
│     for scheme, clspath in handlers.items():                 │
│         self._schemes[scheme] = clspath                      │
│         self._load_handler(scheme, skip_lazy=True)           │
│                                                              │
│  3. 在 _load_handler(skip_lazy=True) 中：                    │
│     - 如果 handler.lazy == True: return None (不实例化)     │
│     - 如果 handler.lazy == False: 实例化并缓存               │
└─────────────────────────────────────────────────────────────┘
```

---

## 8. 各协议处理器详细实现

### 8.1 HTTP/1.1 处理器 (`HTTP11DownloadHandler`)

**文件位置**：`scrapy/core/downloader/handlers/http11.py`

#### 8.1.1 类结构

```
HTTP11DownloadHandler (继承 BaseHttpDownloadHandler)
    │
    ├── ScrapyAgent (封装 Twisted Agent)
    │       ├── _get_agent() - 根据是否有代理选择不同 Agent
    │       │       ├── 无代理: Twisted Agent
    │       │       ├── HTTP 代理: ScrapyProxyAgent
    │       │       └── HTTPS 代理: TunnelingAgent (CONNECT 隧道)
    │       ├── download_request() - 执行下载
    │       ├── _cb_latency() - 计算下载延迟
    │       └── _cb_timeout() - 处理超时
    │
    ├── TunnelingAgent (HTTPS 隧道代理)
    │       └── TunnelingTCP4ClientEndpoint
    │
    └── ScrapyProxyAgent (HTTP 代理)
```

#### 8.1.2 代理处理逻辑

```python
def _get_agent(self, request: Request, timeout: float) -> Agent:
    proxy = request.meta.get("proxy")
    
    if proxy:
        # 有代理的情况
        proxy = add_http_if_no_scheme(proxy)
        proxy_parsed = urlparse(proxy)
        
        if urlparse_cached(request).scheme == "https":
            # HTTPS 通过 CONNECT 隧道
            return self._TunnelingAgent(...)
        else:
            # HTTP 直接使用代理
            return self._ProxyAgent(...)
    
    # 无代理，直接连接
    return self._Agent(...)
```

### 8.2 FTP 处理器 (`FTPDownloadHandler`)

**文件位置**：`scrapy/core/downloader/handlers/ftp.py`

#### 8.2.1 核心特性

- **基于 Twisted FTPClient**：使用 `twisted.protocols.ftp.FTPClient`
- **HTTP 响应模拟**：将 FTP 操作结果封装为 HTTP 响应
- **状态码映射**：

| FTP 状态码 | HTTP 状态码 | 说明 |
|------------|-------------|------|
| 550 | 404 | 文件未找到 |
| 其他 | 503 | 服务不可用 |

#### 8.2.2 配置参数

通过请求 meta 传递连接参数：

- `ftp_user`：用户名（默认 `anonymous`）
- `ftp_password`：密码（默认 `guest`）
- `ftp_passive`：是否使用被动模式（默认 `True`）
- `ftp_local_filename`：下载到本地文件的路径

### 8.3 S3 处理器 (`S3DownloadHandler`)

**文件位置**：`scrapy/core/downloader/handlers/s3.py`

#### 8.3.1 设计模式：包装器 + 委托

```python
class S3DownloadHandler(BaseDownloadHandler):
    lazy = True

    def __init__(self, crawler: Crawler):
        # 1. 检查依赖
        if not is_botocore_available():
            raise NotConfigured("missing botocore library")
        
        super().__init__(crawler)
        
        # 2. 初始化 AWS 签名器
        aws_access_key_id = crawler.settings["AWS_ACCESS_KEY_ID"]
        # ...
        
        # 3. 动态加载当前配置的 HTTPS 处理器
        _http_handler = build_from_crawler(
            load_object(crawler.settings.getwithbase("DOWNLOAD_HANDLERS")["https"]),
            crawler,
        )
        self._download_http = _http_handler.download_request

    async def download_request(self, request: Request) -> Response:
        # 1. 转换 S3 URL 为 HTTP(S) URL
        # s3://bucket/path → https://bucket.s3.amazonaws.com/path
        url = f"{scheme}://{bucket}.s3.amazonaws.com{path}"
        
        # 2. 添加 AWS 签名（如果需要认证）
        if not self.anon:
            # 使用 botocore 签名
            self._signer.add_auth(awsrequest)
            request = request.replace(url=url, headers=awsrequest.headers.items())
        
        # 3. 委托给 HTTP 处理器
        return await self._download_http(request)
```

**关键设计点**：
1. **双重惰性**：类级别 `lazy = True` + 实例化时检查依赖
2. **动态委托**：运行时加载当前配置的 HTTPS 处理器（如果用户配置了 H2，就用 H2）
3. **协议转换**：`s3://` → `http(s)://`

### 8.4 Data URI 处理器 (`DataURIDownloadHandler`)

**文件位置**：`scrapy/core/downloader/handlers/datauri.py`

最简单的处理器，直接解析 `data:` 协议的 URI：

```python
async def download_request(self, request: Request) -> Response:
    uri = parse_data_uri(request.url)
    respcls = responsetypes.from_mimetype(uri.media_type)
    
    if issubclass(respcls, TextResponse) and uri.media_type.split("/")[0] == "text":
        charset = uri.media_type_parameters.get("charset")
        return respcls(url=request.url, body=uri.data, encoding=charset)
    
    return respcls(url=request.url, body=uri.data)
```

### 8.5 文件处理器 (`FileDownloadHandler`)

**文件位置**：`scrapy/core/downloader/handlers/file.py`

处理 `file://` 协议的本地文件：

```python
async def download_request(self, request: Request) -> Response:
    # 1. 转换为本地文件路径
    filepath = file_uri_to_path(request.url)
    
    # 2. 在线程池中读取文件（避免阻塞事件循环）
    body = await run_in_thread(Path(filepath).read_bytes)
    
    # 3. 根据文件类型选择响应类
    respcls = responsetypes.from_args(filename=filepath, body=body)
    return respcls(url=request.url, body=body)
```

---

## 9. 扩展点与自定义

### 9.1 自定义处理器示例

```python
from scrapy.core.downloader.handlers.base import BaseDownloadHandler
from scrapy.http import Response

class MyCustomHandler(BaseDownloadHandler):
    # 设为惰性加载（如果初始化开销大）
    lazy = True
    
    def __init__(self, crawler):
        super().__init__(crawler)
        # 初始化资源
    
    async def download_request(self, request):
        # 实现下载逻辑
        return Response(
            url=request.url,
            status=200,
            body=b"response body"
        )
    
    async def close(self):
        # 清理资源
        pass
```

### 9.2 启用自定义处理器

```python
# settings.py
DOWNLOAD_HANDLERS = {
    # 添加新协议支持
    "myproto": "myproject.handlers.MyCustomHandler",
    # 覆盖现有协议
    "http": "myproject.handlers.MyHttpHandler",
    # 禁用协议
    "ftp": None,
}
```

### 9.3 自定义处理器注意事项

1. **必须实现的方法**：
   - `download_request(self, request) -> Response` (async)
   - `close(self) -> None` (async, 可选但推荐)

2. **必须定义的属性**：
   - `lazy: bool`

3. **异常处理**：
   - 使用 `wrap_twisted_exceptions` 包装 Twisted 异常
   - 或直接抛出 Scrapy 定义的异常

4. **兼容性**：
   - 新风格：返回协程（推荐）
   - 旧风格：返回 `Deferred`，接收 `spider` 参数（已弃用）

---

## 10. 关键事实汇总（复核版）

### 10.1 继承关系修正

| 处理器 | 正确继承关系 | 之前的错误描述 |
|--------|-------------|---------------|
| `HTTP11DownloadHandler` | `BaseHttpDownloadHandler` → `BaseDownloadHandler` | ✅ 正确 |
| `H2DownloadHandler` | `BaseDownloadHandler`（直接） | ❌ 错误地说继承 `BaseHttpDownloadHandler` |
| `FTPDownloadHandler` | `BaseDownloadHandler`（直接） | ✅ 正确 |
| `S3DownloadHandler` | `BaseDownloadHandler`（直接） | ✅ 正确 |
| `DataURIDownloadHandler` | `BaseDownloadHandler`（直接） | ✅ 正确 |
| `FileDownloadHandler` | `BaseDownloadHandler`（直接） | ✅ 正确 |

### 10.2 H2DownloadHandler 不继承 BaseHttpDownloadHandler 的影响

| 特性 | HTTP11DownloadHandler | H2DownloadHandler |
|------|----------------------|-------------------|
| 访问 `self._default_maxsize` | ✅ 直接使用 | ❌ 无此属性 |
| 访问 `self._default_warnsize` | ✅ 直接使用 | ❌ 无此属性 |
| 访问 `self._fail_on_dataloss` | ✅ 直接使用 | ❌ 无此属性 |
| 访问 `self._tls_verbose_logging` | ✅ 直接使用 | ❌ 无此属性 |
| 访问 `self._fail_on_dataloss_warned` | ✅ 直接使用 | ❌ 无此属性 |
| 处理 `stop_download` 信号 | ✅ 使用 `check_stop_download` | ❌ 不支持 |

### 10.3 协议路由关键事实

1. **路由键**：`urlparse_cached(request).scheme`（URL 的协议部分）
2. **配置合并**：`DOWNLOAD_HANDLERS` 覆盖 `DOWNLOAD_HANDLERS_BASE`
3. **禁用方式**：设置为 `None`
4. **路由入口**：`DownloadHandlers.download_request_async()`

### 10.4 惰性加载关键事实

1. **默认值**：`BaseDownloadHandler.lazy = False`（非惰性）
2. **检查时机**：
   - 初始化时：`_load_handler(scheme, skip_lazy=True)`
   - 第一次请求时：`_load_handler(scheme, skip_lazy=False)`
3. **`skip_lazy=True` 时的逻辑**：
   - 如果 `getattr(dhcls, "lazy", True)` → 返回 `None`（不实例化）
   - 注意：未定义 `lazy` 属性时默认 `True`（兼容性）
4. **惰性处理器列表**：`H2DownloadHandler`、`S3DownloadHandler`

### 10.5 HTTP/2 关键事实

1. **ALPN 强制**：通过 `_AcceptableProtocolsContextFactory` 强制协商 `h2`
2. **协议验证**：TLS 握手完成后验证 `negotiatedProtocol == b"h2"`
3. **连接池独立**：`H2ConnectionPool` 支持多路复用和待处理请求队列
4. **流管理**：`Stream` 类独立处理 maxsize，不从 `BaseHttpDownloadHandler` 继承
5. **配置方式**：必须手动设置 `DOWNLOAD_HANDLERS["https"]`

---

## 11. 关键文件索引

| 文件路径 | 功能描述 |
|----------|----------|
| `scrapy/core/downloader/__init__.py` | Downloader 主类，管理下载流程 |
| `scrapy/core/downloader/handlers/__init__.py` | DownloadHandlers 分发器核心 |
| `scrapy/core/downloader/handlers/base.py` | BaseDownloadHandler 抽象基类 |
| `scrapy/utils/_download_handlers.py` | BaseHttpDownloadHandler 中间基类 + 工具函数 |
| `scrapy/core/downloader/handlers/http11.py` | HTTP/1.1 处理器 |
| `scrapy/core/downloader/handlers/http2.py` | HTTP/2 处理器入口 |
| `scrapy/core/http2/agent.py` | HTTP/2 Agent 和连接池 |
| `scrapy/core/http2/protocol.py` | HTTP/2 协议实现 |
| `scrapy/core/http2/stream.py` | HTTP/2 流管理 |
| `scrapy/core/downloader/handlers/ftp.py` | FTP 处理器 |
| `scrapy/core/downloader/handlers/s3.py` | S3 处理器 |
| `scrapy/core/downloader/handlers/datauri.py` | Data URI 处理器 |
| `scrapy/core/downloader/handlers/file.py` | 文件处理器 |
| `scrapy/settings/default_settings.py` | 默认处理器配置 |
| `docs/topics/download-handlers.rst` | 处理器文档 |

---

## 12. 设计亮点总结

### 12.1 架构设计

1. **可插拔架构**：通过配置即可替换或扩展处理器，无需修改核心代码
2. **分层设计**：
   - `Downloader`：管理并发和槽位
   - `DownloadHandlers`：协议分发
   - 具体处理器：协议实现
3. **面向接口编程**：所有处理器遵循 `DownloadHandlerProtocol`

### 12.2 性能优化

1. **惰性加载**：优化启动性能，按需加载处理器
2. **连接池**：HTTP/1.1 和 HTTP/2 都有连接池管理
3. **HTTP/2 多路复用**：单连接多流，减少 TCP 握手开销

### 12.3 灵活性

1. **配置覆盖**：用户配置优先级 > 默认配置
2. **动态委托**：S3 处理器运行时加载当前配置的 HTTPS 处理器
3. **协议无关**：分发器不关心具体协议实现，只关心接口

### 12.4 错误处理

1. **异常标准化**：`wrap_twisted_exceptions` 将 Twisted 异常转换为 Scrapy 异常
2. **隔离失败**：单个处理器初始化失败不影响其他处理器