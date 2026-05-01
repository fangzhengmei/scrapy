# Scrapy 下载处理器多协议分发机制分析报告

## 1. 概述

Scrapy 的下载处理器（Download Handlers）是一套可插拔的组件系统，负责根据请求 URL 的协议（scheme）将请求路由到对应的处理器执行下载任务。本文档深入分析这套多协议分发机制的设计原理、实现细节以及 HTTP/2 支持的集成方式。

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

### 2.3 DownloadHandlers 核心实现

`DownloadHandlers` 类位于 `scrapy/core/downloader/handlers/__init__.py`，是整个分发机制的核心。

#### 2.3.1 数据结构

```python
class DownloadHandlers:
    def __init__(self, crawler: Crawler):
        # 协议到处理器类路径的映射
        self._schemes: dict[str, str | Callable[..., Any]] = {}
        # 已实例化的处理器缓存
        self._handlers: dict[str, DownloadHandlerProtocol] = {}
        # 配置失败的协议及其原因
        self._notconfigured: dict[str, str] = {}
        # 旧风格处理器（返回 Deferred 而非协程）
        self._old_style_handlers: set[str] = set()
```

#### 2.3.2 初始化流程

1. **加载配置**：通过 `crawler.settings.getwithbase("DOWNLOAD_HANDLERS")` 合并用户配置和默认配置
2. **注册协议**：遍历所有协议，将类路径存入 `_schemes`
3. **预加载非惰性处理器**：对于 `lazy=False` 的处理器，提前实例化

#### 2.3.3 请求路由流程

```python
async def download_request_async(self, request: Request) -> Response:
    # 1. 解析 URL 获取协议
    scheme = urlparse_cached(request).scheme
    
    # 2. 获取对应处理器（支持惰性加载）
    handler = self._get_handler(scheme)
    
    if not handler:
        raise NotSupported(
            f"Unsupported URL scheme '{scheme}': {self._notconfigured[scheme]}"
        )
    
    # 3. 调用处理器执行下载
    return await handler.download_request(request)
```

#### 2.3.4 惰性加载机制

```python
def _get_handler(self, scheme: str) -> DownloadHandlerProtocol | None:
    # 1. 检查是否已缓存
    if scheme in self._handlers:
        return self._handlers[scheme]
    
    # 2. 检查是否配置失败
    if scheme in self._notconfigured:
        return None
    
    # 3. 检查是否支持该协议
    if scheme not in self._schemes:
        self._notconfigured[scheme] = "no handler available for that scheme"
        return None
    
    # 4. 惰性实例化处理器
    return self._load_handler(scheme)
```

**惰性加载的优势**：
- 减少启动时间：只在需要时才实例化处理器
- 节省资源：对于不使用的协议（如 S3），不会加载其依赖
- 错误隔离：某个处理器初始化失败不影响其他处理器

### 2.4 处理器接口规范

所有下载处理器必须实现 `DownloadHandlerProtocol` 接口：

```python
class DownloadHandlerProtocol(Protocol):
    # 是否惰性加载
    lazy: bool

    # 执行下载请求
    async def download_request(self, request: Request) -> Response: ...

    # 清理资源
    async def close(self) -> None: ...
```

可选基类 `BaseDownloadHandler` 提供了默认实现：

```python
class BaseDownloadHandler(ABC):
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

## 3. 各协议处理器实现分析

### 3.1 HTTP/1.1 处理器 (`HTTP11DownloadHandler`)

**文件位置**：`scrapy/core/downloader/handlers/http11.py`

#### 3.1.1 核心特性

- **基于 Twisted Agent**：使用 `twisted.web.client.Agent` 作为底层 HTTP 客户端
- **连接池管理**：使用 `HTTPConnectionPool` 实现持久连接
- **代理支持**：支持 HTTP/HTTPS 代理，包括 CONNECT 隧道
- **超时控制**：支持连接超时和读取超时
- **信号集成**：支持 `headers_received` 和 `bytes_received` 信号

#### 3.1.2 类结构

```
HTTP11DownloadHandler (继承 BaseHttpDownloadHandler)
    │
    ├── ScrapyAgent (封装 Twisted Agent)
    │       ├── _get_agent() - 根据是否有代理选择不同 Agent
    │       ├── download_request() - 执行下载
    │       └── 回调处理
    │
    ├── TunnelingAgent (HTTPS 隧道代理)
    │       └── TunnelingTCP4ClientEndpoint
    │
    └── ScrapyProxyAgent (HTTP 代理)
```

#### 3.1.3 代理处理逻辑

```python
def _get_agent(self, request: Request, timeout: float) -> Agent:
    proxy = request.meta.get("proxy")
    
    if proxy:
        # 有代理的情况
        if urlparse_cached(request).scheme == "https":
            # HTTPS 通过 CONNECT 隧道
            return self._TunnelingAgent(...)
        else:
            # HTTP 直接使用代理
            return self._ProxyAgent(...)
    
    # 无代理，直接连接
    return self._Agent(...)
```

### 3.2 FTP 处理器 (`FTPDownloadHandler`)

**文件位置**：`scrapy/core/downloader/handlers/ftp.py`

#### 3.2.1 核心特性

- **基于 Twisted FTPClient**：使用 `twisted.protocols.ftp.FTPClient`
- **HTTP 响应模拟**：将 FTP 操作结果封装为 HTTP 响应
- **状态码映射**：

| FTP 状态码 | HTTP 状态码 | 说明 |
|------------|-------------|------|
| 550 | 404 | 文件未找到 |
| 其他 | 503 | 服务不可用 |

#### 3.2.2 配置参数

通过请求 meta 传递连接参数：

- `ftp_user`：用户名（默认 `anonymous`）
- `ftp_password`：密码（默认 `guest`）
- `ftp_passive`：是否使用被动模式（默认 `True`）
- `ftp_local_filename`：下载到本地文件的路径（大文件下载时避免内存溢出）

### 3.3 S3 处理器 (`S3DownloadHandler`)

**文件位置**：`scrapy/core/downloader/handlers/s3.py`

#### 3.3.1 设计特点

**包装器模式**：S3 处理器本身不执行实际下载，而是：
1. 将 `s3://` URL 转换为标准 HTTP(S) URL
2. 添加 AWS 签名认证
3. 委托给 HTTP 处理器执行下载

```python
async def download_request(self, request: Request) -> Response:
    # 1. 解析 S3 URL
    p = urlparse_cached(request)
    scheme = "https" if request.meta.get("is_secure") else "http"
    bucket = p.hostname
    path = p.path + "?" + p.query if p.query else p.path
    
    # 2. 转换为标准 URL
    url = f"{scheme}://{bucket}.s3.amazonaws.com{path}"
    
    # 3. 添加 AWS 签名（如果需要认证）
    if not self.anon:
        awsrequest = botocore.awsrequest.AWSRequest(...)
        self._signer.add_auth(awsrequest)
        request = request.replace(url=url, headers=awsrequest.headers.items())
    else:
        request = request.replace(url=url)
    
    # 4. 委托给 HTTP 处理器
    return await self._download_http(request)
```

#### 3.3.2 惰性加载

S3 处理器设置 `lazy = True`，只有在实际使用时才会：
- 检查 `botocore` 库是否可用
- 初始化 AWS 签名器

### 3.4 Data URI 处理器 (`DataURIDownloadHandler`)

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

支持格式：
- `data:,A%20brief%20note` - 纯文本
- `data:text/plain;charset=utf-8,Hello` - 指定编码
- `data:image/png;base64,iVBORw0KGgo...` - Base64 编码

### 3.5 文件处理器 (`FileDownloadHandler`)

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

## 4. HTTP/2 支持集成分析

### 4.1 概述

HTTP/2 支持在 Scrapy 中是**实验性功能**，需要：
1. 安装额外依赖：`pip install Twisted[http2]`
2. 手动配置启用

### 4.2 配置方式

```python
DOWNLOAD_HANDLERS = {
    "https": "scrapy.core.downloader.handlers.http2.H2DownloadHandler",
}
```

**注意**：HTTP/2 处理器只支持 `https` 协议，不支持明文 HTTP/2（h2c），因为主流浏览器都不支持未加密的 HTTP/2。

### 4.3 模块架构

HTTP/2 实现分布在多个模块中：

```
scrapy/core/downloader/handlers/http2.py
    └── H2DownloadHandler (处理器入口)
    └── ScrapyH2Agent (代理封装)

scrapy/core/http2/
    ├── agent.py
    │   ├── H2ConnectionPool (连接池)
    │   ├── H2Agent (核心 Agent)
    │   └── ScrapyProxyH2Agent (代理支持)
    │
    ├── protocol.py
    │   ├── H2ClientProtocol (H2 协议实现)
    │   └── H2ClientFactory (协议工厂)
    │
    └── stream.py
        └── Stream (单个 HTTP/2 流)
```

### 4.4 H2DownloadHandler 实现

```python
class H2DownloadHandler(BaseHttpDownloadHandler):
    lazy = True  # 惰性加载

    def __init__(self, crawler: Crawler):
        # 检查 Twisted reactor 是否启用
        if not crawler.settings.getbool("TWISTED_REACTOR_ENABLED"):
            raise NotConfigured(f"{type(self).__name__} requires a Twisted reactor.")
        
        super().__init__(crawler)
        
        # HTTP/2 连接池
        self._pool = H2ConnectionPool(reactor, crawler.settings)
        
        # TLS 上下文工厂（强制使用 h2 协议）
        self._context_factory = _load_context_factory_from_settings(crawler)

    async def download_request(self, request: Request) -> Response:
        agent = ScrapyH2Agent(
            context_factory=self._context_factory,
            pool=self._pool,
            bind_address=self._bind_address,
            crawler=self._crawler,
        )
        
        with wrap_twisted_exceptions():
            return await maybe_deferred_to_future(
                agent.download_request(request, self._crawler.spider)
            )
```

### 4.5 连接池管理 (`H2ConnectionPool`)

HTTP/2 的连接池与 HTTP/1.1 不同，因为：
- **多路复用**：单个连接可以承载多个并发请求
- **连接键**：使用 `(scheme, host, port)` 作为键

```python
class H2ConnectionPool:
    def __init__(self, reactor: ReactorBase, settings: Settings):
        # 活跃连接：key -> H2ClientProtocol
        self._connections: dict[ConnectionKeyT, H2ClientProtocol] = {}
        
        # 待处理请求：在连接建立前到达的请求
        self._pending_requests: dict[
            ConnectionKeyT, deque[Deferred[H2ClientProtocol]]
        ] = {}

    def get_connection(self, key: ConnectionKeyT, uri: URI, endpoint: HostnameEndpoint):
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

### 4.6 协议实现 (`H2ClientProtocol`)

`H2ClientProtocol` 基于 `hyper-h2` 库实现，负责：
- 管理 HTTP/2 连接状态
- 处理多路复用流
- 处理帧的收发

```python
class H2ClientProtocol(Protocol, TimeoutMixin):
    IDLE_TIMEOUT = 240

    def __init__(self, uri: URI, settings: Settings, ...):
        # hyper-h2 连接实例
        config = H2Configuration(client_side=True, header_encoding="utf-8")
        self.conn = H2Connection(config=config)
        
        # 流 ID 生成器（奇数，客户端发起）
        self._stream_id_generator = itertools.count(start=1, step=2)
        
        # 流管理：stream_id -> Stream
        self.streams: dict[int, Stream] = {}
        
        # 待发送请求池
        self._pending_request_stream_pool: deque[Stream] = deque()
```

#### 4.6.1 ALPN 协议协商

通过 `_AcceptableProtocolsContextFactory` 强制协商 `h2` 协议：

```python
class _AcceptableProtocolsContextFactory:
    def __init__(self, context_factory, acceptable_protocols=[b"h2"]):
        self._context_factory = context_factory
        self._acceptable_protocols = acceptable_protocols

    def creatorForNetloc(self, hostname, port):
        context = self._context_factory.creatorForNetloc(hostname, port)
        # 设置 ALPN 协议
        context.set_alpn_protocols(self._acceptable_protocols)
        # 设置 NPN 协议（向后兼容）
        context.set_npn_protocols(self._acceptable_protocols)
        return context
```

如果服务器不支持 HTTP/2，连接会失败并抛出 `InvalidNegotiatedProtocol` 异常。

### 4.7 流管理 (`Stream`)

每个 HTTP/2 请求对应一个 `Stream` 对象，负责：
- 发送请求头和数据
- 接收响应头和数据
- 处理流量控制

```python
class Stream:
    def __init__(self, stream_id: int, request: Request, protocol: H2ClientProtocol, ...):
        self.stream_id = stream_id
        self._request = request
        self._protocol = protocol
        
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

#### 4.7.1 多路复用调度

```python
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

### 4.8 HTTP/2 与 HTTP/1.1 的对比

| 特性 | HTTP/1.1 | HTTP/2 |
|------|----------|--------|
| 默认处理器 | ✅ 是 | ❌ 否，需手动配置 |
| 连接复用 | 每个请求一个连接（或有限持久连接） | 单个连接多路复用 |
| 并发模型 | 依赖多个 TCP 连接 | 单连接多流 |
| 头部压缩 | ❌ 无 | ✅ HPACK |
| 服务器推送 | ❌ 不支持 | ✅ 支持（但 Scrapy 忽略） |
| 代理支持 | ✅ 完整 | ⚠️ 部分（不支持 HTTPS 隧道） |
| 信号支持 | ✅ headers_received, bytes_received | ❌ 不支持 |

### 4.9 已知限制

根据文档，HTTP/2 处理器存在以下限制：

1. **不支持明文 HTTP/2 (h2c)**：因为主流浏览器都不支持未加密的 HTTP/2
2. **不支持更大帧大小**：连接到发送大于默认 16384 字节帧的服务器会失败
3. **服务器推送被忽略**：虽然 HTTP/2 支持服务器推送，但 Scrapy 会忽略这些推送
4. **不支持部分信号**：`bytes_received` 和 `headers_received` 信号不可用
5. **HTTPS 隧道代理不支持**：无法通过 CONNECT 方法使用 HTTP/2 隧道

## 5. 跨模块协作分析

### 5.1 整体调用链

```
请求进入
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│                    Downloader.fetch()                         │
│  scrapy/core/downloader/__init__.py:126                      │
│  - 管理下载槽位 (Slot)                                        │
│  - 处理并发控制                                                │
│  - 通过中间件链                                                │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│            DownloaderMiddlewareManager.download_async()      │
│  - 执行下载中间件                                              │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│              Downloader._enqueue_request()                    │
│  scrapy/core/downloader/__init__.py:176                      │
│  - 入队请求到槽位                                              │
│  - 触发队列处理                                                │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                Downloader._download()                         │
│  scrapy/core/downloader/__init__.py:221                      │
│  - 标记请求为传输中                                            │
│  - 调用处理器执行下载                                          │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│       DownloadHandlers.download_request_async()              │
│  scrapy/core/downloader/handlers/__init__.py:141            │
│  - 解析 URL 协议                                              │
│  - 路由到对应处理器                                            │
│  - 惰性加载处理器                                              │
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

### 5.2 模块依赖关系

```
scrapy.core.downloader.Downloader
    │
    ├── scrapy.core.downloader.handlers.DownloadHandlers (核心分发)
    │       │
    │       ├── scrapy.core.downloader.handlers.http11.HTTP11DownloadHandler
    │       │       ├── scrapy.utils._download_handlers.BaseHttpDownloadHandler
    │       │       ├── twisted.web.client.Agent
    │       │       └── twisted.web.client.HTTPConnectionPool
    │       │
    │       ├── scrapy.core.downloader.handlers.http2.H2DownloadHandler
    │       │       ├── scrapy.utils._download_handlers.BaseHttpDownloadHandler
    │       │       ├── scrapy.core.http2.agent.H2Agent
    │       │       │       ├── scrapy.core.http2.agent.H2ConnectionPool
    │       │       │       └── scrapy.core.http2.protocol.H2ClientProtocol
    │       │       │               ├── scrapy.core.http2.stream.Stream
    │       │       │               └── h2.connection.H2Connection (hyper-h2)
    │       │       └── scrapy.core.downloader.contextfactory._AcceptableProtocolsContextFactory
    │       │
    │       ├── scrapy.core.downloader.handlers.ftp.FTPDownloadHandler
    │       │       ├── scrapy.core.downloader.handlers.base.BaseDownloadHandler
    │       │       └── twisted.protocols.ftp.FTPClient
    │       │
    │       ├── scrapy.core.downloader.handlers.s3.S3DownloadHandler
    │       │       ├── scrapy.core.downloader.handlers.base.BaseDownloadHandler
    │       │       ├── botocore.auth (AWS 签名)
    │       │       └── [委托给 HTTP 处理器]
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

### 5.3 共享组件

#### 5.3.1 `BaseHttpDownloadHandler`

位于 `scrapy/utils/_download_handlers.py`，为 HTTP 相关处理器提供共享配置：

```python
class BaseHttpDownloadHandler(BaseDownloadHandler, ABC):
    def __init__(self, crawler: Crawler):
        super().__init__(crawler)
        self._default_maxsize: int = crawler.settings.getint("DOWNLOAD_MAXSIZE")
        self._default_warnsize: int = crawler.settings.getint("DOWNLOAD_WARNSIZE")
        self._fail_on_dataloss: bool = crawler.settings.getbool("DOWNLOAD_FAIL_ON_DATALOSS")
        self._tls_verbose_logging: bool = crawler.settings.getbool(
            "DOWNLOADER_CLIENT_TLS_VERBOSE_LOGGING"
        )
```

#### 5.3.2 `wrap_twisted_exceptions`

将 Twisted 异常转换为 Scrapy 标准异常：

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
    # ... 更多异常映射
```

#### 5.3.3 `make_response`

构建标准 Scrapy 响应对象：

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
    # ...
```

### 5.4 配置加载流程

```
┌─────────────────────────────────────────────────────────────┐
│  用户 settings.py                                             │
│  DOWNLOAD_HANDLERS = {                                        │
│      "https": "scrapy.core.downloader.handlers.http2.H2..." │
│  }                                                            │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  BaseSettings.getwithbase("DOWNLOAD_HANDLERS")              │
│  - 合并 DOWNLOAD_HANDLERS 和 DOWNLOAD_HANDLERS_BASE          │
│  - 用户配置优先级 > 默认配置                                   │
│  - 值为 None 表示禁用该协议                                    │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  DownloadHandlers.__init__()                                 │
│  - 过滤掉值为 None 的协议                                      │
│  - 注册到 self._schemes                                       │
│  - 预加载非惰性处理器 (lazy=False)                            │
└─────────────────────────────────────────────────────────────┘
```

## 6. 扩展点与自定义

### 6.1 自定义处理器示例

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

### 6.2 启用自定义处理器

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

## 7. 总结

### 7.1 设计亮点

1. **可插拔架构**：通过配置即可替换或扩展处理器
2. **惰性加载**：优化启动性能，按需加载
3. **统一接口**：所有处理器遵循相同的 `DownloadHandlerProtocol`
4. **分层设计**：
   - `Downloader`：管理并发和槽位
   - `DownloadHandlers`：协议分发
   - 具体处理器：协议实现
5. **异常标准化**：通过 `wrap_twisted_exceptions` 统一异常类型

### 7.2 HTTP/2 集成特点

1. **完全兼容现有分发机制**：HTTP/2 处理器只是另一个处理器实现，无需修改核心分发逻辑
2. **独立的连接池**：`H2ConnectionPool` 针对 HTTP/2 多路复用特性设计
3. **ALPN 强制协商**：通过 `_AcceptableProtocolsContextFactory` 确保只协商 `h2` 协议
4. **实验性功能**：需要手动配置启用，有已知限制

### 7.3 关键文件索引

| 文件路径 | 功能描述 |
|----------|----------|
| `scrapy/core/downloader/__init__.py` | Downloader 主类，管理下载流程 |
| `scrapy/core/downloader/handlers/__init__.py` | DownloadHandlers 分发器核心 |
| `scrapy/core/downloader/handlers/http11.py` | HTTP/1.1 处理器 |
| `scrapy/core/downloader/handlers/http2.py` | HTTP/2 处理器入口 |
| `scrapy/core/http2/agent.py` | HTTP/2 Agent 和连接池 |
| `scrapy/core/http2/protocol.py` | HTTP/2 协议实现 |
| `scrapy/core/http2/stream.py` | HTTP/2 流管理 |
| `scrapy/core/downloader/handlers/ftp.py` | FTP 处理器 |
| `scrapy/core/downloader/handlers/s3.py` | S3 处理器 |
| `scrapy/core/downloader/handlers/datauri.py` | Data URI 处理器 |
| `scrapy/core/downloader/handlers/file.py` | 文件处理器 |
| `scrapy/utils/_download_handlers.py` | HTTP 处理器基类和工具函数 |
| `scrapy/settings/default_settings.py` | 默认处理器配置 |
| `docs/topics/download-handlers.rst` | 处理器文档 |
