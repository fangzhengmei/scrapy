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

DOWNLOAD_HANDLERS = {}  # 用户配置，默认为空
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

## 3. 配置合并与禁用协议的真实生效环节（修正版）

### 3.1 配置合并的真实实现

**之前的描述**：使用了简化的伪代码逻辑

**正确事实**：`getwithbase()` 方法实际创建新的 `BaseSettings` 对象并执行两次 `update()`

#### 3.1.1 真实代码实现

```python
# scrapy/settings/__init__.py:325
def getwithbase(self, name: _SettingsKey) -> BaseSettings:
    """Get a composition of a dictionary-like setting and its ``_BASE``
    counterpart.
    """
    if not isinstance(name, str):
        raise ValueError(f"Base setting key must be a string, got {name}")
    
    # 创建新的 BaseSettings 对象
    compbs = BaseSettings()
    
    # 第一步：先更新默认配置
    compbs.update(self[name + "_BASE"])
    
    # 第二步：再更新用户配置（覆盖默认）
    compbs.update(self[name])
    
    return compbs
```

#### 3.1.2 配置合并流程图

```
┌─────────────────────────────────────────────────────────────┐
│  配置合并流程（getwithbase("DOWNLOAD_HANDLERS")）           │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  1. 创建新的 BaseSettings 对象 (compbs)                      │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  2. 第一次 update: compbs.update(self["DOWNLOAD_HANDLERS_BASE"]) │
│                                                              │
│  此时 compbs 内容：                                          │
│  {                                                           │
│    "http": "HTTP11DownloadHandler",                         │
│    "https": "HTTP11DownloadHandler",                        │
│    "ftp": "FTPDownloadHandler",                             │
│    "data": "DataURIDownloadHandler",                        │
│    "file": "FileDownloadHandler",                           │
│    "s3": "S3DownloadHandler"                                │
│  }                                                           │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  3. 第二次 update: compbs.update(self["DOWNLOAD_HANDLERS"]) │
│                                                              │
│  假设用户配置：                                              │
│  {                                                           │
│    "ftp": None,                    # 禁用 FTP               │
│    "https": "H2DownloadHandler"    # 覆盖 HTTPS 处理器    │
│  }                                                           │
│                                                              │
│  此时 compbs 内容：                                          │
│  {                                                           │
│    "http": "HTTP11DownloadHandler",     # 保留默认         │
│    "https": "H2DownloadHandler",        # 用户覆盖         │
│    "ftp": None,                       # 被设为 None        │
│    "data": "DataURIDownloadHandler",  # 保留默认         │
│    "file": "FileDownloadHandler",      # 保留默认         │
│    "s3": "S3DownloadHandler"          # 保留默认         │
│  }                                                           │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 禁用协议的真实生效环节

**关键发现**：禁用协议的生效不是在 `getwithbase()` 中，而是在 `DownloadHandlers.__init__()` 中的 `without_none_values()` 调用

#### 3.2.1 真实生效流程

```python
# scrapy/core/downloader/handlers/__init__.py:60
def __init__(self, crawler: Crawler):
    # ...
    
    # 关键：without_none_values 会过滤掉值为 None 的键值对
    handlers: dict[str, str | Callable[..., Any]] = without_none_values(
        cast(
            "dict[str, str | Callable[..., Any]]",
            crawler.settings.getwithbase("DOWNLOAD_HANDLERS"),
        )
    )
    
    # 只有过滤后的协议才会被注册
    for scheme, clspath in handlers.items():
        self._schemes[scheme] = clspath
        self._load_handler(scheme, skip_lazy=True)
```

#### 3.2.2 `without_none_values` 实现

```python
# scrapy/utils/python.py:257
def without_none_values(
    iterable: Mapping[_KT, _VT] | Iterable[_KT],
) -> dict[_KT, _VT] | Iterable[_VT]:
    """Return a copy of ``iterable`` with all ``None`` entries removed.
    
    If ``iterable`` is a mapping, return a dictionary where all pairs that have
    value ``None`` have been removed.
    """
    if isinstance(iterable, Mapping):
        # 关键：值为 None 的键值对不会被保留
        return {k: v for k, v in iterable.items() if v is not None}
    
    # 其他类型的处理...
    return type(iterable)(v for v in iterable if v is not None)
```

#### 3.2.3 完整禁用流程图

```
┌─────────────────────────────────────────────────────────────┐
│  用户设置: DOWNLOAD_HANDLERS = {"ftp": None, "https": "H2Handler"} │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  1. getwithbase("DOWNLOAD_HANDLERS")                        │
│     → 合并后的结果:                                           │
│     {                                                        │
│       "http": "HTTP11Handler",                              │
│       "https": "H2Handler",      # 用户覆盖                │
│       "ftp": None,               # 被设为 None             │
│       "data": "DataURIHandler",                            │
│       "file": "FileHandler",                               │
│       "s3": "S3Handler"                                   │
│     }                                                        │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  2. without_none_values(merged_config)                      │
│     → 过滤掉值为 None 的键值对                               │
│     → 结果:                                                  │
│     {                                                        │
│       "http": "HTTP11Handler",                              │
│       "https": "H2Handler",                                 │
│       # "ftp" 被移除了！                                    │
│       "data": "DataURIHandler",                            │
│       "file": "FileHandler",                               │
│       "s3": "S3Handler"                                   │
│     }                                                        │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  3. 注册到 self._schemes                                      │
│     → self._schemes 内容:                                    │
│     {                                                        │
│       "http": "HTTP11Handler",                              │
│       "https": "H2Handler",                                 │
│       "data": "DataURIHandler",                            │
│       "file": "FileHandler",                               │
│       "s3": "S3Handler"                                    │
│       # 注意：没有 "ftp"！                                  │
│     }                                                        │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  4. 请求 "ftp://example.com/file" 时                         │
│     → _get_handler("ftp")                                   │
│     → "ftp" not in self._schemes                            │
│     → self._notconfigured["ftp"] = "no handler available for that scheme" │
│     → 抛出 NotSupported 异常                                │
└─────────────────────────────────────────────────────────────┘
```

### 3.3 配置合并与禁用协议关键事实汇总

| 环节 | 之前的描述 | 正确事实 |
|------|-----------|---------|
| 配置合并 | 简化伪代码 | `getwithbase()` 创建新 `BaseSettings`，先 `update(_BASE)` 再 `update(用户配置)` |
| 禁用协议生效 | 在 `getwithbase()` 中过滤 | 在 `DownloadHandlers.__init__()` 中的 `without_none_values()` 过滤 |
| 禁用标记 | 值为 `None` 的协议被移除 | 值为 `None` 的协议不会被注册到 `self._schemes` |
| 请求时检查 | 检查 `self._handlers` | 首先检查 `self._schemes`，未注册则记录到 `self._notconfigured` |

---

## 4. 继承关系与职责定位（修正版）

### 4.1 继承层次结构

**关键修正 1**：`H2DownloadHandler` 直接继承 `BaseDownloadHandler`，**不是** `BaseHttpDownloadHandler`

**新增发现**：`HttpxDownloadHandler` 继承 `BaseHttpDownloadHandler`，用于非 Twisted 运行模式

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
┌─────────────────────┐ ┌─────────────────────┐
│ HTTP11Downloader    │ │ HttpxDownloader     │
│                     │ │  (实验性)           │
│  lazy = False       │ │                     │
│  检查 TWISTED_      │ │  lazy = False       │
│  REACTOR_ENABLED    │ │  不检查 TWISTED_   │
│                     │ │  REACTOR_ENABLED    │
└─────────────────────┘ └─────────────────────┘

其他直接继承 BaseDownloadHandler 的处理器：
┌─────────────────────┐ ┌─────────────────────┐
│ DataURIDownloader   │ │ FileDownloader      │
│                     │ │                     │
│  lazy = False       │ │  lazy = False       │
│  不依赖 Twisted     │ │  不依赖 Twisted     │
└─────────────────────┘ └─────────────────────┘
```

### 4.2 各基类职责分析

#### 4.2.1 `BaseDownloadHandler` - 最底层抽象基类

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

#### 4.2.2 `BaseHttpDownloadHandler` - HTTP 特定中间基类

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

### 4.3 各处理器继承关系汇总表

| 处理器 | 直接父类 | 间接父类 | lazy 值 | 检查 TWISTED_REACTOR_ENABLED | 使用 HTTP 特定配置 |
|--------|----------|----------|---------|------------------------------|-------------------|
| `HTTP11DownloadHandler` | `BaseHttpDownloadHandler` | `BaseDownloadHandler` | `False` | ✅ 是 | ✅ 是 |
| `H2DownloadHandler` | `BaseDownloadHandler` | 无 | `True` | ✅ 是 | ❌ 否（独立实现） |
| `HttpxDownloadHandler` | `BaseHttpDownloadHandler` | `BaseDownloadHandler` | `False` | ❌ 否（检查 `is_asyncio_available()`） | ✅ 是 |
| `FTPDownloadHandler` | `BaseDownloadHandler` | 无 | `False` | ✅ 是 | ❌ 否 |
| `S3DownloadHandler` | `BaseDownloadHandler` | 无 | `True` | ❌ 否 | ❌ 否 |
| `DataURIDownloadHandler` | `BaseDownloadHandler` | 无 | `False` | ❌ 否 | ❌ 否 |
| `FileDownloadHandler` | `BaseDownloadHandler` | 无 | `False` | ❌ 否 | ❌ 否 |

### 4.4 关键修正：H2DownloadHandler 的特殊设计

**之前的错误描述**：`H2DownloadHandler` 继承 `BaseHttpDownloadHandler`

**正确事实**：`H2DownloadHandler` 直接继承 `BaseDownloadHandler`，**不继承** `BaseHttpDownloadHandler`

#### 4.4.1 代码证据

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

对比 `HttpxDownloadHandler`（用于非 Twisted 模式）：

```python
# scrapy/core/downloader/handlers/_httpx.py:75
class HttpxDownloadHandler(BaseHttpDownloadHandler):  # 继承 BaseHttpDownloadHandler
    _DEFAULT_CONNECT_TIMEOUT = 10

    def __init__(self, crawler: Crawler):
        # 不检查 TWISTED_REACTOR_ENABLED，而是检查 is_asyncio_available()
        if not is_asyncio_available():
            raise NotConfigured(
                f"{type(self).__name__} requires the asyncio support. Make"
                f" sure that you have either enabled the asyncio Twisted"
                f" reactor in the TWISTED_REACTOR setting or disabled the"
                f" TWISTED_REACTOR_ENABLED setting."
            )
        if httpx is None:
            raise NotConfigured(
                f"{type(self).__name__} requires the httpx library to be installed."
            )
        super().__init__(crawler)  # 调用 BaseHttpDownloadHandler.__init__
        # 可以使用 self._default_maxsize, self._default_warnsize 等
```

---

## 5. TLS 协商时的对象层级与上下文工厂构建链路（深度挖掘版）

### 5.1 上下文工厂体系结构

Scrapy 的 TLS 上下文工厂采用**包装器模式**（Wrapper Pattern），支持多层包装。

```
┌─────────────────────────────────────────────────────────────────┐
│                    IPolicyForHTTPS (接口)                        │
│  twisted/web/iweb.py                                              │
│                                                                   │
│  方法:                                                            │
│  - creatorForNetloc(hostname: bytes, port: int) → ClientTLSOptions │
└───────────────────────────┬─────────────────────────────────────┘
                            │
            ┌───────────────┼───────────────┐
            │               │               │
            ▼               ▼               ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│ BrowserLikePolicy │ │ _ScrapyClient   │ │ _Acceptable     │
│ ForHTTPS (Twisted)│ │ ContextFactory  │ │ ProtocolsContext │
│                   │ │                 │ │ Factory         │
│ (默认安全策略)     │ │ (Scrapy 扩展)   │ │ (包装器)         │
└─────────────────┘ └────────┬────────┘ └────────┬────────┘
                             │                    │
                             └─────────┬──────────┘
                                       │
                                       ▼
                              ┌─────────────────┐
                              │ 包装关系:        │
                              │                 │
                              │ _Acceptable     │
                              │ ProtocolsContext │
                              │ Factory         │
                              │     │           │
                              │     │ 包装器    │
                              │     ▼           │
                              │ _ScrapyClient   │
                              │ ContextFactory  │
                              │     │           │
                              │     │ 继承      │
                              │     ▼           │
                              │ BrowserLikePolicy │
                              │ ForHTTPS        │
                              └─────────────────┘
```

### 5.2 HTTP/1.1 处理器的上下文工厂构建链路

#### 5.2.1 构建流程

```
┌─────────────────────────────────────────────────────────────┐
│  HTTP11DownloadHandler.__init__(crawler)                     │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  1. _load_context_factory_from_settings(crawler)             │
│                                                              │
│     scrapy/core/downloader/contextfactory.py:228           │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  2. 检查 DOWNLOADER_CLIENTCONTEXTFACTORY 设置                │
│                                                              │
│     if settings["DOWNLOADER_CLIENTCONTEXTFACTORY"] == "SENTINEL": │
│         context_factory_cls = _ScrapyClientContextFactory   │
│     else:                                                     │
│         context_factory_cls = load_object(...)  # 用户自定义 │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  3. build_from_crawler(context_factory_cls, crawler)         │
│                                                              │
│     → 调用 _ScrapyClientContextFactory.from_crawler(crawler) │
│     → 创建 _ScrapyClientContextFactory 实例                  │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  4. 返回 _ScrapyClientContextFactory 实例                    │
│                                                              │
│     HTTP11DownloadHandler 直接使用这个实例                   │
│     没有额外的包装器层                                       │
└─────────────────────────────────────────────────────────────┘
```

#### 5.2.2 实际使用时的调用链

当建立 TLS 连接时：

```
┌─────────────────────────────────────────────────────────────┐
│  Twisted Agent 需要建立 TLS 连接                              │
│  目标: https://example.com:443                               │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  1. 调用 context_factory.creatorForNetloc(b"example.com", 443) │
│                                                              │
│     这里的 context_factory 是 _ScrapyClientContextFactory   │
│     (没有被包装)                                             │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  2. _ScrapyClientContextFactory.creatorForNetloc()          │
│                                                              │
│     if not self._verify_certificates:                        │
│         # 返回 _ScrapyClientTLSOptions（跳过证书验证）       │
│         return _ScrapyClientTLSOptions(...)                 │
│     else:                                                    │
│         # 使用 Twisted 的 optionsForClientTLS               │
│         return optionsForClientTLS(...)                     │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  3. 返回 ClientTLSOptions 实例                               │
│                                                              │
│     Twisted 使用这个实例中的 SSL 上下文建立 TLS 连接         │
└─────────────────────────────────────────────────────────────┘
```

### 5.3 HTTP/2 处理器的上下文工厂构建链路（关键修正）

**关键发现**：HTTP/2 的上下文工厂有**两层**：
1. 基础层：`_ScrapyClientContextFactory`（从设置加载）
2. 包装层：`_AcceptableProtocolsContextFactory`（强制设置 ALPN 协议）

#### 5.3.1 构建流程图

```
┌─────────────────────────────────────────────────────────────┐
│  H2DownloadHandler.__init__(crawler)                        │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  1. _load_context_factory_from_settings(crawler)             │
│                                                              │
│     → 返回 _ScrapyClientContextFactory 实例                 │
│     (这一步与 HTTP/1.1 相同)                                 │
│                                                              │
│     self._context_factory = 这个实例                        │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  2. 后续在 H2Agent 中进行包装                                │
│                                                              │
│     注意：包装不是在 H2DownloadHandler 中完成的，            │
│           而是在 ScrapyH2Agent 内部的 H2Agent 中完成的     │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  3. H2Agent.__init__() 中的包装                             │
│                                                              │
│     scrapy/core/http2/agent.py                              │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  4. 使用 _AcceptableProtocolsContextFactory 进行包装         │
│                                                              │
│     self._context_factory = _AcceptableProtocolsContextFactory( │
│         context_factory,           # 被包装的对象：_ScrapyClientContextFactory │
│         acceptable_protocols=[b"h2"]  # 强制协商的协议     │
│     )                                                        │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  5. 最终的上下文工厂结构                                      │
│                                                              │
│     _AcceptableProtocolsContextFactory (包装器)              │
│         │                                                    │
│         └── _wrapped_context_factory: _ScrapyClientContextFactory │
│                                                              │
│     同时：                                                    │
│     self._acceptable_protocols = [b"h2"]                   │
└─────────────────────────────────────────────────────────────┘
```

#### 5.3.2 实际使用时的调用链

当建立 TLS 连接时，HTTP/2 的调用链与 HTTP/1.1 有本质区别：

```
┌─────────────────────────────────────────────────────────────┐
│  H2ConnectionPool 需要建立 TLS 连接                          │
│  目标: https://example.com:443 (HTTP/2)                     │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  1. 调用 context_factory.creatorForNetloc(b"example.com", 443) │
│                                                              │
│     这里的 context_factory 是 _AcceptableProtocolsContextFactory │
│     (是包装器，不是原始的 _ScrapyClientContextFactory)       │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  2. _AcceptableProtocolsContextFactory.creatorForNetloc()   │
│                                                              │
│     scrapy/core/downloader/contextfactory.py:212            │
│                                                              │
│     def creatorForNetloc(self, hostname, port):            │
│         # 第一步：调用被包装的上下文工厂                     │
│         options = self._wrapped_context_factory.creatorForNetloc(
│             hostname, port
│         )                                                    │
│                                                              │
│         # 第二步：强制设置 ALPN 协议                         │
│         # _setAcceptableProtocols 来自 twisted.internet._sslverify │
│         _setAcceptableProtocols(options._ctx, self._acceptable_protocols)
│                                                              │
│         # 第三步：返回修改后的 options                        │
│         return options                                       │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  3. 内部调用 _ScrapyClientContextFactory.creatorForNetloc()  │
│                                                              │
│     → 返回 ClientTLSOptions 实例                             │
│     → 这个实例包含了基本的 SSL 上下文                        │
│     → 但还没有设置 ALPN 协议                                 │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  4. 调用 _setAcceptableProtocols(options._ctx, [b"h2"])     │
│                                                              │
│     这个函数会：                                              │
│     - 设置 SSL 上下文的 ALPN 协议列表                        │
│     - 设置 NPN 协议列表（向后兼容）                           │
│                                                              │
│     结果：SSL 上下文现在只能协商 "h2" 协议                   │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  5. 返回修改后的 ClientTLSOptions 实例                       │
│                                                              │
│     Twisted 使用这个实例建立 TLS 连接                        │
│     TLS 握手时会使用 ALPN 协商 "h2" 协议                     │
└─────────────────────────────────────────────────────────────┘
```

### 5.4 _setAcceptableProtocols 函数作用

这个函数来自 Twisted，用于设置 ALPN/NPN 协议：

```python
# 伪代码逻辑
def _setAcceptableProtocols(ctx, acceptable_protocols):
    """设置 SSL 上下文的 ALPN 和 NPN 协议列表"""
    
    # 设置 ALPN 协议（现代 TLS 握手使用）
    ctx.set_alpn_protocols(acceptable_protocols)  # e.g., [b"h2"]
    
    # 设置 NPN 协议（向后兼容，用于旧版 TLS）
    ctx.set_npn_protocols(acceptable_protocols)
```

### 5.5 HTTP/1.1 vs HTTP/2 上下文工厂对比

| 特性 | HTTP/1.1 | HTTP/2 |
|------|----------|--------|
| 基础上下文工厂 | `_ScrapyClientContextFactory` | `_ScrapyClientContextFactory` |
| 包装层 | ❌ 无 | ✅ `_AcceptableProtocolsContextFactory` |
| ALPN 协议设置 | 未强制设置（使用默认） | 强制设置为 `[b"h2"]` |
| 协商结果 | 可以协商 http/1.1 或其他 | 只能协商 h2 |
| 协议验证 | 无 | TLS 握手后验证 `negotiatedProtocol == b"h2"` |

### 5.6 TLS 协商完整对象层级图

```
┌─────────────────────────────────────────────────────────────────────┐
│                      HTTP/2 TLS 协商对象层级                         │
└───────────────────────────────────┬─────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│  H2Agent                                                            │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ self._context_factory = _AcceptableProtocolsContextFactory( │   │
│  │     context_factory=_ScrapyClientContextFactory(...),       │   │
│  │     acceptable_protocols=[b"h2"]                            │   │
│  │ )                                                             │   │
│  └─────────────────────────────────────────────────────────────┘   │
└───────────────────────────────────┬─────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│  _AcceptableProtocolsContextFactory (包装器)                         │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 属性:                                                         │   │
│  │   - _wrapped_context_factory: _ScrapyClientContextFactory   │   │
│  │   - _acceptable_protocols: [b"h2"]                          │   │
│  │                                                               │   │
│  │ 方法:                                                         │   │
│  │   - creatorForNetloc(hostname, port):                       │   │
│  │       1. options = self._wrapped_context_factory.creatorForNetloc(...) │
│  │       2. _setAcceptableProtocols(options._ctx, [b"h2"])    │   │
│  │       3. return options                                      │   │
│  └─────────────────────────────────────────────────────────────┘   │
└───────────────────────────────────┬─────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│  _ScrapyClientContextFactory (被包装的实际实现)                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 属性:                                                         │   │
│  │   - _ssl_method: int (TLS 方法)                              │   │
│  │   - tls_verbose_logging: bool                                │   │
│  │   - tls_ciphers: AcceptableCiphers                           │   │
│  │   - _verify_certificates: bool                               │   │
│  │                                                               │   │
│  │ 方法:                                                         │   │
│  │   - creatorForNetloc(hostname, port):                       │   │
│  │       → 返回 ClientTLSOptions 实例                           │   │
│  │       → 包含基本的 SSL 上下文配置                            │   │
│  └─────────────────────────────────────────────────────────────┘   │
└───────────────────────────────────┬─────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│  TLS 握手时的实际调用                                                 │
│                                                                       │
│  1. H2Agent 需要建立连接                                             │
│  2. 调用 self._context_factory.creatorForNetloc(b"example.com", 443) │
│  3. 这实际上是 _AcceptableProtocolsContextFactory.creatorForNetloc() │
│  4. 内部调用 _wrapped_context_factory.creatorForNetloc()            │
│  5. 得到 ClientTLSOptions 实例，其 _ctx 包含基本 SSL 配置           │
│  6. 调用 _setAcceptableProtocols(options._ctx, [b"h2"])            │
│  7. options._ctx 现在强制设置了 ALPN 协议为 [b"h2"]                 │
│  8. Twisted 使用这个 options 建立 TLS 连接                           │
│  9. TLS 握手时，ALPN 协议协商只能是 "h2"                             │
│ 10. 如果服务器不支持 h2，协商会失败，连接会被关闭                    │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.7 HTTP/2 协议验证（握手后）

即使 ALPN 协商成功，HTTP/2 处理器还会在 TLS 握手完成后进行额外验证：

```python
# scrapy/core/http2/protocol.py

class H2ClientProtocol(Protocol, TimeoutMixin):
    # ...
    
    def handshakeCompleted(self) -> None:
        """TLS 握手完成后的回调"""
        assert self.transport is not None
        
        # 验证协商的协议是否为 h2
        if (
            self.transport.negotiatedProtocol is not None
            and self.transport.negotiatedProtocol != PROTOCOL_NAME  # PROTOCOL_NAME = b"h2"
        ):
            # 协议不匹配，关闭连接
            self._lose_connection_with_error(
                [InvalidNegotiatedProtocol(self.transport.negotiatedProtocol)]
            )
```

**这意味着**：
1. 即使 ALPN 协商设置正确
2. 如果服务器实际协商的不是 `h2`（而是 `http/1.1`）
3. 连接会被立即关闭
4. 抛出 `InvalidNegotiatedProtocol` 异常

---

## 6. 非 Twisted 运行模式下的默认协议分发行为变化（新增深度分析）

### 6.1 TWISTED_REACTOR_ENABLED 概述

**默认值**：`TWISTED_REACTOR_ENABLED = True`（`scrapy/settings/default_settings.py:531`）

**设置为 `False` 时**：Scrapy 不使用 Twisted reactor，而是使用纯 asyncio 事件循环

### 6.2 各处理器对 TWISTED_REACTOR_ENABLED 的检查

| 处理器 | 检查方式 | 非 Twisted 模式下行为 |
|--------|----------|---------------------|
| `HTTP11DownloadHandler` | `getbool("TWISTED_REACTOR_ENABLED")` | 抛出 `NotConfigured` |
| `H2DownloadHandler` | `getbool("TWISTED_REACTOR_ENABLED")` | 抛出 `NotConfigured` |
| `FTPDownloadHandler` | `getbool("TWISTED_REACTOR_ENABLED")` | 抛出 `NotConfigured` |
| `HttpxDownloadHandler` | `is_asyncio_available()` | 如果有 asyncio 循环则正常工作 |
| `S3DownloadHandler` | 不检查（但需要 HTTP 处理器） | 依赖的 HTTP 处理器不可用 |
| `DataURIDownloadHandler` | 不检查 | 正常工作 |
| `FileDownloadHandler` | 不检查 | 正常工作 |

### 6.3 处理器检查代码证据

#### 6.3.1 HTTP11DownloadHandler

```python
# scrapy/core/downloader/handlers/http11.py:85
class HTTP11DownloadHandler(BaseHttpDownloadHandler):
    def __init__(self, crawler: Crawler):
        if not crawler.settings.getbool("TWISTED_REACTOR_ENABLED"):
            raise NotConfigured(f"{type(self).__name__} requires a Twisted reactor.")
        super().__init__(crawler)
        # ...
```

#### 6.3.2 FTPDownloadHandler

```python
# scrapy/core/downloader/handlers/ftp.py:88
class FTPDownloadHandler(BaseDownloadHandler):
    def __init__(self, crawler: Crawler):
        if not crawler.settings.getbool("TWISTED_REACTOR_ENABLED"):
            raise NotConfigured(f"{type(self).__name__} requires a Twisted reactor.")
        super().__init__(crawler)
        # ...
```

#### 6.3.3 HttpxDownloadHandler（特殊）

```python
# scrapy/core/downloader/handlers/_httpx.py:80
class HttpxDownloadHandler(BaseHttpDownloadHandler):
    def __init__(self, crawler: Crawler):
        # 不检查 TWISTED_REACTOR_ENABLED
        # 而是检查 is_asyncio_available()
        if not is_asyncio_available():
            raise NotConfigured(
                f"{type(self).__name__} requires the asyncio support. Make"
                f" sure that you have either enabled the asyncio Twisted"
                f" reactor in the TWISTED_REACTOR setting or disabled the"
                f" TWISTED_REACTOR_ENABLED setting."
            )
        if httpx is None:
            raise NotConfigured(
                f"{type(self).__name__} requires the httpx library to be installed."
            )
        super().__init__(crawler)
        # ...
```

#### 6.3.4 DataURIDownloadHandler（无检查）

```python
# scrapy/core/downloader/handlers/datauri.py:15
class DataURIDownloadHandler(BaseDownloadHandler):
    async def download_request(self, request: Request) -> Response:
        # 没有任何 TWISTED_REACTOR_ENABLED 检查
        # 直接解析 data: URI
        uri = parse_data_uri(request.url)
        # ...
```

### 6.4 非 Twisted 模式下的默认处理器状态

**默认配置下**（`TWISTED_REACTOR_ENABLED = False` 且用户未配置 `DOWNLOAD_HANDLERS`）：

| 协议 | 默认处理器 | 非 Twisted 模式下状态 |
|------|-----------|---------------------|
| `http` | `HTTP11DownloadHandler` | ❌ 不可用（抛出 `NotConfigured`） |
| `https` | `HTTP11DownloadHandler` | ❌ 不可用（抛出 `NotConfigured`） |
| `ftp` | `FTPDownloadHandler` | ❌ 不可用（抛出 `NotConfigured`） |
| `s3` | `S3DownloadHandler` | ❌ 不可用（依赖的 HTTPS 处理器不可用） |
| `data` | `DataURIDownloadHandler` | ✅ 正常工作 |
| `file` | `FileDownloadHandler` | ✅ 正常工作 |

### 6.5 非 Twisted 模式下的请求处理流程

```
┌─────────────────────────────────────────────────────────────┐
│  请求: "http://example.com/page"                             │
│  模式: TWISTED_REACTOR_ENABLED = False                       │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  1. DownloadHandlers.download_request_async()               │
│     → scheme = "http"                                       │
│     → _get_handler("http")                                  │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  2. _get_handler("http")                                     │
│     → "http" in self._schemes? → Yes                        │
│     → "http" in self._handlers? → No (首次请求)            │
│     → 调用 _load_handler("http", skip_lazy=False)          │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  3. _load_handler("http")                                    │
│     → dhcls = HTTP11DownloadHandler                         │
│     → dh = build_from_crawler(dhcls, crawler)              │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  4. HTTP11DownloadHandler.__init__(crawler)                 │
│     → if not crawler.settings.getbool("TWISTED_REACTOR_ENABLED"): │
│         raise NotConfigured("HTTP11DownloadHandler requires a Twisted reactor.") │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  5. _load_handler 捕获 NotConfigured 异常                    │
│     → self._notconfigured["http"] = "HTTP11DownloadHandler requires a Twisted reactor." │
│     → return None                                           │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  6. _get_handler 返回 None                                   │
│     → 抛出 NotSupported: "Unsupported URL scheme 'http': HTTP11DownloadHandler requires a Twisted reactor." │
└─────────────────────────────────────────────────────────────┘
```

### 6.6 非 Twisted 模式下使用 HTTP 的配置方式

**用户必须手动配置 `HttpxDownloadHandler`** 才能在非 Twisted 模式下使用 HTTP/HTTPS：

```python
# settings.py
TWISTED_REACTOR_ENABLED = False

DOWNLOAD_HANDLERS = {
    "http": "scrapy.core.downloader.handlers._httpx.HttpxDownloadHandler",
    "https": "scrapy.core.downloader.handlers._httpx.HttpxDownloadHandler",
}
```

### 6.7 HttpxDownloadHandler 与其他处理器的对比

| 特性 | HTTP11DownloadHandler | HttpxDownloadHandler |
|------|----------------------|---------------------|
| 继承关系 | `BaseHttpDownloadHandler` | `BaseHttpDownloadHandler` |
| 依赖网络库 | Twisted | httpx |
| 检查 TWISTED_REACTOR_ENABLED | ✅ 是 | ❌ 否 |
| 检查 is_asyncio_available | ❌ 否 | ✅ 是 |
| 支持代理 | ✅ 完整 | ❌ 不支持请求级别代理 |
| 支持绑定地址 | ✅ 完整 | ⚠️ 仅支持主机，不支持端口 |
| 支持信号 | ✅ `bytes_received`, `headers_received` | ✅ `bytes_received`, `headers_received` |
| 文档状态 | 稳定 | 实验性 |

### 6.8 非 Twisted 模式测试用例证据

```python
# tests/AsyncCrawlerRunner/reactorless_datauri.py
# 这是一个非 Twisted 模式下的测试

import asyncio

from scrapy import Request, Spider
from scrapy.crawler import AsyncCrawlerRunner
from scrapy.utils.log import configure_logging


class DataSpider(Spider):
    name = "data"

    async def start(self):
        yield Request("data:,foo")  # 只有 data: URI 可以正常工作

    def parse(self, response):
        return {"data": response.text}


async def main() -> None:
    configure_logging()
    runner = AsyncCrawlerRunner(settings={"TWISTED_REACTOR_ENABLED": False})
    await runner.crawl(DataSpider)


asyncio.run(main())
```

**测试用例说明**：
- 只测试 `data:` URI 协议
- 不测试 `http://` 或 `https://` 协议
- 这说明默认配置下非 Twisted 模式无法使用 HTTP

### 6.9 非 Twisted 模式关键事实汇总

| 事实 | 说明 |
|------|------|
| 默认 HTTP 处理器不可用 | `HTTP11DownloadHandler` 检查 `TWISTED_REACTOR_ENABLED`，为 `False` 时抛出 `NotConfigured` |
| 必须手动配置 `HttpxDownloadHandler` | 这是目前非 Twisted 模式下使用 HTTP 的唯一方式 |
| `DataURIDownloadHandler` 和 `FileDownloadHandler` 正常工作 | 它们不依赖 Twisted |
| `HttpxDownloadHandler` 继承 `BaseHttpDownloadHandler` | 可以使用 `_default_maxsize` 等属性，支持信号 |
| `HttpxDownloadHandler` 是实验性的 | 文档标记为 "not recommended for production" |

---

## 7. 协议路由机制（复核版）

### 7.1 核心实现

`DownloadHandlers` 类位于 `scrapy/core/downloader/handlers/__init__.py`，是整个分发机制的核心。

#### 7.1.1 数据结构

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

#### 7.1.2 初始化流程

```python
def __init__(self, crawler: Crawler):
    # 1. 合并配置：DOWNLOAD_HANDLERS + DOWNLOAD_HANDLERS_BASE
    # 2. 过滤值为 None 的协议（禁用的协议）
    handlers: dict[str, str | Callable[..., Any]] = without_none_values(
        cast(
            "dict[str, str | Callable[..., Any]]",
            crawler.settings.getwithbase("DOWNLOAD_HANDLERS"),
        )
    )
    
    # 3. 遍历所有协议，注册到 _schemes
    for scheme, clspath in handlers.items():
        self._schemes[scheme] = clspath
        # 4. 尝试预加载（非惰性处理器会被实例化）
        self._load_handler(scheme, skip_lazy=True)
    
    # 5. 注册关闭信号
    crawler.signals.connect(self._close, signals.engine_stopped)
```

#### 7.1.3 请求路由流程

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

### 7.2 _get_handler 完整逻辑

```python
def _get_handler(self, scheme: str) -> DownloadHandlerProtocol | None:
    """Lazy-load the downloadhandler for a scheme
    only on the first request for that scheme.
    """
    # 1. 检查是否已缓存实例
    if scheme in self._handlers:
        return self._handlers[scheme]
    
    # 2. 检查是否配置失败
    if scheme in self._notconfigured:
        return None
    
    # 3. 检查是否支持该协议（是否已注册到 self._schemes）
    if scheme not in self._schemes:
        # 如果没有注册，记录到 _notconfigured
        self._notconfigured[scheme] = "no handler available for that scheme"
        return None
    
    # 4. 尝试加载（实例化）处理器
    return self._load_handler(scheme)
```

### 7.3 协议分发完整流程图

```
┌─────────────────────────────────────────────────────────────┐
│  请求: "https://example.com/page"                            │
│  模式: TWISTED_REACTOR_ENABLED = True                        │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  1. urlparse_cached(request).scheme                          │
│     → "https"                                                │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  2. _get_handler("https")                                    │
└───────────────────────────┬─────────────────────────────────┘
                            │
            ┌───────────────┼───────────────┐
            │               │               │
            ▼               ▼               ▼
┌───────────────┐ ┌───────────────┐ ┌───────────────┐
│ 已缓存?       │ │ 配置失败?     │ │ 已注册?       │
│               │ │               │ │               │
│ "https" in    │ │ "https" in    │ │ "https" in    │
│ self._handlers│ │ self._notconfig│ │ self._schemes │
└───────┬───────┘ └───────┬───────┘ └───────┬───────┘
        │                 │                 │
        ▼                 ▼                 ▼
     返回实例          返回 None         继续检查
                                            │
                                            ▼
                                    ┌───────────────┐
                                    │ 调用          │
                                    │ _load_handler │
                                    └───────┬───────┘
                                            │
                                            ▼
                                    ┌───────────────┐
                                    │ 实例化处理器   │
                                    │ 检查依赖      │
                                    │ 缓存到        │
                                    │ self._handlers│
                                    └───────┬───────┘
                                            │
                                            ▼
                                    ┌───────────────┐
                                    │ 返回处理器实例 │
                                    └───────────────┘
```

---

## 8. 惰性加载机制（复核版）

### 8.1 设计目的

惰性加载（Lazy Loading）的设计目的：

1. **减少启动时间**：只在需要时才实例化处理器
2. **节省资源**：对于不使用的协议（如 S3），不会加载其依赖
3. **错误隔离**：某个处理器初始化失败不影响其他处理器

### 8.2 实现机制

#### 8.2.1 `lazy` 属性

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

#### 8.2.2 `_load_handler` 方法核心逻辑

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

### 8.3 加载时序图

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

### 8.4 惰性加载处理器列表

| 处理器 | lazy 值 | 原因 |
|--------|----------|------|
| `H2DownloadHandler` | `True` | 实验性功能，默认不启用 |
| `S3DownloadHandler` | `True` | 需要 `botocore` 依赖，可能未安装 |
| `HTTP11DownloadHandler` | `False` | 默认 HTTP 处理器，总是需要 |
| `FTPDownloadHandler` | `False` | 默认协议处理器 |
| `DataURIDownloadHandler` | `False` | 轻量级，无依赖 |
| `FileDownloadHandler` | `False` | 轻量级，无依赖 |
| `HttpxDownloadHandler` | `False` | 未指定，使用默认值 |

### 8.5 S3 处理器的特殊惰性行为

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

## 9. HTTP/2 支持集成分析（复核版）

### 9.1 概述

HTTP/2 支持在 Scrapy 中是**实验性功能**，具有以下特点：

1. **默认未启用**：需要手动配置
2. **独立实现**：有自己的连接池、协议、流管理
3. **仅支持 HTTPS**：不支持明文 HTTP/2 (h2c)
4. **惰性加载**：`lazy = True`
5. **强制 ALPN 协商**：通过 `_AcceptableProtocolsContextFactory` 包装器

### 9.2 配置方式

```python
# settings.py
DOWNLOAD_HANDLERS = {
    "https": "scrapy.core.downloader.handlers.http2.H2DownloadHandler",
}
```

**重要**：HTTP/2 处理器只支持 `https` 协议，不支持 `http` 协议。

### 9.3 模块架构

```
scrapy/core/downloader/handlers/http2.py
    │
    ├── H2DownloadHandler (继承 BaseDownloadHandler)
    │   ├── lazy = True
    │   ├── __init__(crawler)
    │   │   ├── 检查 TWISTED_REACTOR_ENABLED
    │   │   ├── 创建 H2ConnectionPool
    │   │   └── 加载 TLS 上下文工厂（基础层）
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
    │   ├── H2Agent (核心 Agent)
    │   │   ├── 关键：使用 _AcceptableProtocolsContextFactory 包装
    │   │   │       上下文工厂
    │   │   └── endpoint_factory
    │   └── ScrapyProxyH2Agent (HTTP 代理支持)
    │
    ├── protocol.py
    │   ├── H2ClientProtocol (协议实现)
    │   │   ├── handshakeCompleted() - 验证协商协议
    │   │   └── conn: H2Connection (hyper-h2)
    │   └── H2ClientFactory (协议工厂)
    │
    └── stream.py
        └── Stream (单个 HTTP/2 流)
```

### 9.