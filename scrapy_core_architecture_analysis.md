# Scrapy 爬虫框架核心调度架构分析报告

> 分析版本：基于 Scrapy 源码（2026-04-28 分析）

## 目录

1. [架构概览](#1-架构概览)
2. [引擎协调机制](#2-引擎协调机制)
3. [调度器队列管理策略](#3-调度器队列管理策略)
4. [Crawler 对象整合机制](#4-crawler-对象整合机制)
5. [核心配置与扩展系统](#5-核心配置与扩展系统)

---

## 1. 架构概览

### 1.1 核心组件层次结构

Scrapy 采用分层架构设计，核心组件从上到下依次为：

```
┌─────────────────────────────────────────────────────────────┐
│                     Crawler (顶层整合者)                      │
│  ┌───────────┬───────────┬───────────┬───────────────────┐ │
│  │ Extension │  Stats    │  Signals  │   AddonManager    │ │
│  │  Manager  │ Collector │  Manager  │                   │ │
│  └───────────┴───────────┴───────────┴───────────────────┘ │
└───────────────────────────────┬─────────────────────────────┘
                                │
┌───────────────────────────────▼─────────────────────────────┐
│                  ExecutionEngine (执行引擎)                   │
│  ┌──────────────┬──────────────┬──────────────────────────┐ │
│  │  Scheduler   │  Downloader  │        Scraper           │ │
│  │  (调度器)     │  (下载器)    │        (解析器)           │ │
│  └──────────────┴──────────────┴──────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 数据流循环

Scrapy 的核心爬取循环遵循经典的生产者-消费者模式：

```
┌──────────────────────────────────────────────────────────────────┐
│                         爬取循环 (Crawl Loop)                      │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│   ┌──────────┐     ┌──────────┐     ┌──────────┐                │
│   │ Scheduler│────▶│Downloader│────▶│ Scraper  │                │
│   │ (取请求)  │     │ (下载)   │     │ (解析)   │                │
│   └──────────┘     └──────────┘     └────┬─────┘                │
│       ▲                                    │                       │
│       │                                    ▼                       │
│       │         ┌──────────────────────────────────┐              │
│       │         │    产出: Item 或 新 Request      │              │
│       │         │  - Item → Item Pipeline 处理      │              │
│       └─────────┤  - Request → 返回 Scheduler 队列  │              │
│                 └──────────────────────────────────┘              │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

---

## 2. 引擎协调机制

### 2.1 ExecutionEngine 核心职责

`ExecutionEngine` 是 Scrapy 的核心协调者，位于 `scrapy/core/engine.py`，负责：

1. **组件生命周期管理**：初始化和关闭调度器、下载器、解析器
2. **请求流协调**：驱动 "取请求→下载→解析" 的完整循环
3. **并发控制**：通过 backout 机制防止系统过载
4. **状态管理**：跟踪爬虫运行状态（running/paused/stopping）

### 2.2 核心循环驱动机制

引擎通过 `_start_scheduled_requests` 方法驱动爬取循环：

```python
# engine.py:329-338
def _start_scheduled_requests(self) -> None:
    if self._slot is None or self._slot.closing is not None or self.paused:
        return

    while not self.needs_backout():
        if not self._start_scheduled_request():
            break

    if self.spider_is_idle() and self._slot.close_if_idle:
        self._spider_idle()
```

**循环流程解析**：

| 步骤 | 方法 | 职责 | 文件位置 |
|------|------|------|----------|
| 1 | `needs_backout()` | 检查是否需要限流（并发控制） | engine.py:340-353 |
| 2 | `_start_scheduled_request()` | 从调度器取请求并触发下载 | engine.py:355-395 |
| 3 | `spider_is_idle()` | 检查是否无待处理请求 | engine.py:421-430 |
| 4 | `_spider_idle()` | 处理空闲状态（可能关闭爬虫） | engine.py:559-582 |

### 2.3 请求下载与响应处理链路

#### 2.3.1 下载流程

```python
# engine.py:483-518
@inlineCallbacks
def _download(
    self, request: Request
) -> Generator[Deferred[Any], Any, Response | Request]:
    # 1. 记录请求到 slot（进行中状态）
    self._slot.add_request(request)
    try:
        # 2. 调用下载器执行实际下载
        result = yield self.downloader.fetch(request)
        # 3. 处理响应（记录日志、发送信号）
        if isinstance(result, Response):
            logkws = self.logformatter.crawled(result.request, result, self.spider)
            self.signals.send_catch_log(
                signal=signals.response_received,
                response=result,
                request=result.request,
                spider=self.spider,
            )
        return result
    finally:
        # 4. 调度下一轮处理
        self._slot.nextcall.schedule()
```

#### 2.3.2 响应解析流程

下载完成后，响应通过 `_handle_downloader_output` 传递给 Scraper：

```python
# engine.py:397-419
@inlineCallbacks
def _handle_downloader_output(
    self, result: Request | Response | Failure, request: Request
) -> Generator[Deferred[Any], Any, None]:
    # 下载器中间件可能返回新请求（如重定向）
    if isinstance(result, Request):
        self.crawl(result)  # 重新入队
        return

    try:
        # 交给 Scraper 解析
        yield self.scraper.enqueue_scrape(result, request)
    except Exception:
        logger.error("Error while enqueuing scrape", exc_info=True)
```

### 2.4 背压控制 (Backpressure)

引擎通过 `needs_backout()` 方法实现流控：

```python
# engine.py:340-353
def needs_backout(self) -> bool:
    return (
        not self.running
        or not self._slot
        or bool(self._slot.closing)
        or self.downloader.needs_backout()      # 下载器并发限制
        or self.scraper.slot.needs_backout()   # 解析器内存限制
    )
```

**限流条件**：

| 组件 | 限流条件 | 配置项 |
|------|----------|--------|
| Downloader | `len(active) >= CONCURRENT_REQUESTS` | `CONCURRENT_REQUESTS` (默认16) |
| Scraper | `active_size > SCRAPER_SLOT_MAX_ACTIVE_SIZE` | `SCRAPER_SLOT_MAX_ACTIVE_SIZE` (默认5MB) |

### 2.5 Slot 机制

引擎使用 `_Slot` 类管理单个爬虫的运行状态：

```python
# engine.py:64-98
class _Slot:
    def __init__(self, close_if_idle: bool, nextcall: CallLaterOnce, scheduler: BaseScheduler):
        self.closing: Deferred[None] | None = None
        self.inprogress: set[Request] = set()  # 进行中的请求
        self.close_if_idle: bool = close_if_idle
        self.nextcall: CallLaterOnce[None] = nextcall  # 调度下次处理
        self.scheduler: BaseScheduler = scheduler
        self.heartbeat: AsyncioLoopingCall | LoopingCall  # 心跳机制（5秒间隔）
```

**心跳机制**：每 5 秒调用 `nextcall.schedule()`，确保即使调度器报告有请求但返回 None 时也能继续尝试。

---

## 3. 调度器队列管理策略

### 3.1 Scheduler 核心架构

`Scheduler` 位于 `scrapy/core/scheduler.py`，负责请求的存储和分发。

#### 3.1.1 核心属性

```python
# scheduler.py:270-330
class Scheduler(BaseScheduler):
    def __init__(self, ...):
        self.df: BaseDupeFilter = dupefilter              # 去重过滤器
        self.dqdir: str | None = self._dqdir(jobdir)     # 磁盘队列目录
        self.pqclass: type[ScrapyPriorityQueue]           # 优先级队列类
        self.dqclass: type[BaseQueue] | None              # 磁盘队列类
        self.mqclass: type[BaseQueue] | None              # 内存队列类
        self.mqs: ScrapyPriorityQueue                      # 内存优先级队列
        self.dqs: ScrapyPriorityQueue | None               # 磁盘优先级队列
```

#### 3.1.2 内存/磁盘双队列架构

```
┌─────────────────────────────────────────────────────────────────┐
│                         Scheduler                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              mqs (内存优先级队列)                         │   │
│  │  ┌─────────┬─────────┬─────────┬─────────┐             │   │
│  │  │ prio=-2 │ prio=-1 │ prio=0  │ prio=1  │             │   │
│  │  │ (最高)  │         │         │ (最低)  │             │   │
│  │  └─────────┴─────────┴─────────┴─────────┘             │   │
│  └─────────────────────────────────────────────────────────┘   │
│                           ▲                                       │
│                           │ 优先出队                               │
│  ┌────────────────────────┼──────────────────────────────────┐   │
│  │                        │                                  │   │
│  │  ┌─────────────────────▼──────────────────────────────┐  │   │
│  │  │           dqs (磁盘优先级队列 - 可选)               │  │   │
│  │  │  ┌─────────┬─────────┬─────────┬─────────┐        │  │   │
│  │  │  │ prio=-2 │ prio=-1 │ prio=0  │ prio=1  │        │  │   │
│  │  │  └─────────┴─────────┴─────────┴─────────┘        │  │   │
│  │  │                                                      │  │   │
│  │  │  持久化位置: JOBDIR/requests.queue/                 │  │   │
│  │  └──────────────────────────────────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.2 入队策略

```python
# scheduler.py:367-388
def enqueue_request(self, request: Request) -> bool:
    # 1. 去重检查
    if not request.dont_filter and self.df.request_seen(request):
        self.df.log(request, self.spider)
        return False
    
    # 2. 先尝试入磁盘队列
    dqok = self._dqpush(request)
    if dqok:
        self.stats.inc_value("scheduler/enqueued/disk")
    else:
        # 3. 磁盘失败则入内存队列（如不可序列化的请求）
        self._mqpush(request)
        self.stats.inc_value("scheduler/enqueued/memory")
    
    self.stats.inc_value("scheduler/enqueued")
    return True
```

#### 磁盘队列入队逻辑

```python
# scheduler.py:417-439
def _dqpush(self, request: Request) -> bool:
    if self.dqs is None:
        return False  # 无磁盘队列时直接失败
    try:
        self.dqs.push(request)
    except ValueError as e:  # 不可序列化的请求
        if self.logunser:
            logger.warning("Unable to serialize request...")
            self.logunser = False
        self.stats.inc_value("scheduler/unserializable")
        return False
    return True
```

### 3.3 出队策略

```python
# scheduler.py:390-409
def next_request(self) -> Request | None:
    # 1. 优先从内存队列取
    request: Request | None = self.mqs.pop()
    if request is not None:
        self.stats.inc_value("scheduler/dequeued/memory")
    else:
        # 2. 内存空了才从磁盘队列取
        request = self._dqpop()
        if request is not None:
            self.stats.inc_value("scheduler/dequeued/disk")
    
    if request is not None:
        self.stats.inc_value("scheduler/dequeued")
    return request
```

### 3.4 磁盘队列启用条件

磁盘队列 **仅在配置 `JOBDIR` 时启用**：

```python
# scheduler.py:514-521
def _dqdir(self, jobdir: str | None) -> str | None:
    """Return a folder name to keep disk queue state at"""
    if jobdir:
        dqdir = Path(jobdir, "requests.queue")
        if not dqdir.exists():
            dqdir.mkdir(parents=True)
        return str(dqdir)
    return None
```

**初始化时创建队列**：

```python
# scheduler.py:345-354
def open(self, spider: Spider) -> Deferred[None] | None:
    self.spider: Spider = spider
    self.mqs: ScrapyPriorityQueue = self._mq()           # 始终创建内存队列
    self.dqs: ScrapyPriorityQueue | None = self._dq() if self.dqdir else None  # 可选磁盘队列
    return self.df.open()
```

### 3.5 优先级队列实现

#### 3.5.1 ScrapyPriorityQueue

位于 `scrapy/pqueues.py`，为每个优先级维护独立的子队列：

```python
# pqueues.py:169-182
def push(self, request: Request) -> None:
    priority = self.priority(request)  # priority = -request.priority
    is_start_request = request.meta.get("is_start_request", False)
    
    # 起始请求有独立队列
    if is_start_request and self._start_queue_cls:
        if priority not in self._start_queues:
            self._start_queues[priority] = self._sqfactory(priority)
        q = self._start_queues[priority]
    else:
        if priority not in self.queues:
            self.queues[priority] = self.qfactory(priority)
        q = self.queues[priority]
    
    q.push(request)
    # 更新当前最高优先级
    if self.curprio is None or priority < self.curprio:
        self.curprio = priority
```

**优先级规则**：
- `priority = -request.priority`（负数表示高优先级）
- 数字越小优先级越高（`priority=-2` > `priority=-1` > `priority=0`）

#### 3.5.2 DownloaderAwarePriorityQueue

增强版优先级队列，考虑下载器负载：

```python
# pqueues.py:277-280
class DownloaderAwarePriorityQueue:
    """PriorityQueue which takes Downloader activity into account:
    domains (slots) with the least amount of active downloads are dequeued
    first.
    """
```

**调度策略**：
- 为每个下载 slot（域名/IP）维护独立的 `ScrapyPriorityQueue`
- 出队时选择 **活动下载数最少** 的 slot
- 实现公平调度，避免单个域名阻塞其他域名

### 3.6 队列类型配置

默认队列配置（`default_settings.py`）：

```python
# default_settings.py:481-487
SCHEDULER = "scrapy.core.scheduler.Scheduler"
SCHEDULER_DISK_QUEUE = "scrapy.squeues.PickleLifoDiskQueue"      # 磁盘 LIFO
SCHEDULER_MEMORY_QUEUE = "scrapy.squeues.LifoMemoryQueue"        # 内存 LIFO
SCHEDULER_PRIORITY_QUEUE = "scrapy.pqueues.DownloaderAwarePriorityQueue"
SCHEDULER_START_DISK_QUEUE = "scrapy.squeues.PickleFifoDiskQueue"  # 起始请求 FIFO
SCHEDULER_START_MEMORY_QUEUE = "scrapy.squeues.FifoMemoryQueue"    # 起始请求 FIFO
```

#### 队列类型对比

| 队列类 | 存储 | 顺序 | 序列化 | 适用场景 |
|--------|------|------|--------|----------|
| `LifoMemoryQueue` | 内存 | LIFO (栈) | 否 | 默认内存队列，DFS 遍历 |
| `FifoMemoryQueue` | 内存 | FIFO (队列) | 否 | BFS 遍历 |
| `PickleLifoDiskQueue` | 磁盘 | LIFO | Pickle | 默认磁盘队列 |
| `PickleFifoDiskQueue` | 磁盘 | FIFO | Pickle | BFS 持久化 |
| `MarshalLifoDiskQueue` | 磁盘 | LIFO | Marshal | 更快的序列化 |

### 3.7 断点续传机制

当爬虫使用 `JOBDIR` 时，调度器支持断点续传：

```python
# scheduler.py:356-365
def close(self, reason: str) -> Deferred[None] | None:
    # 1. 保存磁盘队列状态到 active.json
    if self.dqs is not None:
        state = self.dqs.close()
        self._write_dqs_state(self.dqdir, state)
    return self.df.close(reason)

# scheduler.py:476-512
def _dq(self) -> ScrapyPriorityQueue:
    # 恢复时读取之前保存的状态
    state = self._read_dqs_state(self.dqdir)
    q = build_from_crawler(
        self.pqclass, self.crawler,
        downstream_queue_cls=self.dqclass,
        key=self.dqdir,
        startprios=state,  # 恢复优先级状态
        start_queue_cls=self._sdqclass,
    )
    if q:
        logger.info("Resuming crawl (%(queuesize)d requests scheduled)", ...)
    return q
```

**状态文件**：`JOBDIR/requests.queue/active.json` 存储活跃的优先级列表。

---

## 4. Crawler 对象整合机制

### 4.1 Crawler 核心职责

`Crawler` 是 Scrapy 的顶层整合者，位于 `scrapy/crawler.py`，负责：

1. **组件装配**：创建并协调所有核心组件
2. **配置管理**：加载和合并配置
3. **扩展系统**：加载扩展、中间件、管道
4. **信号系统**：提供事件发布订阅机制
5. **生命周期管理**：控制爬虫的启动和停止

### 4.2 Crawler 组件结构

```python
# crawler.py:56-86
class Crawler:
    def __init__(self, spidercls: type[Spider], settings: ...):
        self.spidercls: type[Spider] = spidercls
        self.settings: Settings = settings.copy()
        
        # 核心管理器
        self.addons: AddonManager = AddonManager(self)
        self.signals: SignalManager = SignalManager(self)
        
        # 延迟初始化的组件
        self.extensions: ExtensionManager | None = None
        self.stats: StatsCollector | None = None
        self.logformatter: LogFormatter | None = None
        self.request_fingerprinter: RequestFingerprinterProtocol | None = None
        self.spider: Spider | None = None
        self.engine: ExecutionEngine | None = None
```

### 4.3 组件初始化流程

#### 4.3.1 启动流程

```python
# crawler.py:199-227
async def crawl_async(self, *args: Any, **kwargs: Any) -> None:
    # 1. 创建 Spider 实例
    self.spider = self._create_spider(*args, **kwargs)
    
    # 2. 应用配置并初始化所有组件
    self._apply_settings()
    
    # 3. 创建执行引擎
    self.engine = self._create_engine()
    
    # 4. 打开 Spider（初始化调度器等）
    await self.engine.open_spider_async()
    
    # 5. 启动引擎主循环
    await self.engine.start_async()
```

#### 4.3.2 组件装配 (`_apply_settings`)

```python
# crawler.py:93-151
def _apply_settings(self) -> None:
    if self.settings.frozen:
        return
    
    # 1. 加载 Addon 配置
    self.addons.load_settings(self.settings)
    
    # 2. 初始化统计收集器
    self.stats = load_object(self.settings["STATS_CLASS"])(self)
    
    # 3. 初始化日志格式化器
    lf_cls = load_object(self.settings["LOG_FORMATTER"])
    self.logformatter = lf_cls.from_crawler(self)
    
    # 4. 初始化请求指纹生成器
    self.request_fingerprinter = build_from_crawler(
        load_object(self.settings["REQUEST_FINGERPRINTER_CLASS"]), self
    )
    
    # 5. 配置 Reactor（Twisted/Asyncio）
    use_reactor = self.settings.getbool("TWISTED_REACTOR_ENABLED")
    if use_reactor:
        # 安装/验证 Reactor
        ...
        log_reactor_info()
    else:
        # 无 Reactor 模式（使用纯 Asyncio）
        self._apply_reactorless_default_settings()
    
    # 6. 加载扩展管理器
    self.extensions = ExtensionManager.from_crawler(self)
    
    # 7. 冻结配置（防止后续修改）
    self.settings.freeze()
```

### 4.4 引擎创建

```python
# crawler.py:232-233
def _create_engine(self) -> ExecutionEngine:
    return ExecutionEngine(self, lambda _: self.stop_async())
```

**引擎回调机制**：
- 第二个参数是 `spider_closed_callback`
- 当爬虫关闭时，引擎会调用此回调触发 `Crawler.stop_async()`

### 4.5 组件获取 API

Crawler 提供统一的组件获取接口：

```python
# crawler.py:265-340
def get_addon(self, cls: type[_T]) -> _T | None:
    return self._get_component(cls, self.addons.addons)

def get_downloader_middleware(self, cls: type[_T]) -> _T | None:
    return self._get_component(cls, self.engine.downloader.middleware.middlewares)

def get_extension(self, cls: type[_T]) -> _T | None:
    return self._get_component(cls, self.extensions.middlewares)

def get_item_pipeline(self, cls: type[_T]) -> _T | None:
    return self._get_component(cls, self.engine.scraper.itemproc.middlewares)

def get_spider_middleware(self, cls: type[_T]) -> _T | None:
    return self._get_component(cls, self.engine.scraper.spidermw.middlewares)
```

---

## 5. 核心配置与扩展系统

### 5.1 扩展系统架构

#### 5.1.1 ExtensionManager

扩展管理器继承自 `MiddlewareManager`：

```python
# extension.py:18-24
class ExtensionManager(MiddlewareManager):
    component_name = "extension"

    @classmethod
    def _get_mwlist_from_settings(cls, settings: Settings) -> list[Any]:
        return build_component_list(
            settings.get_component_priority_dict_with_base("EXTENSIONS")
        )
```

#### 5.1.2 默认扩展

```python
# default_settings.py:311-323
EXTENSIONS_BASE = {
    "scrapy.extensions.corestats.CoreStats": 0,           # 核心统计
    "scrapy.extensions.logcount.LogCount": 0,              # 日志计数
    "scrapy.extensions.telnet.TelnetConsole": 0,           # Telnet 控制台
    "scrapy.extensions.memusage.MemoryUsage": 0,           # 内存使用监控
    "scrapy.extensions.memdebug.MemoryDebugger": 0,        # 内存调试
    "scrapy.extensions.closespider.CloseSpider": 0,        # 自动关闭
    "scrapy.extensions.feedexport.FeedExporter": 0,        # 数据导出
    "scrapy.extensions.logstats.LogStats": 0,               # 日志统计
    "scrapy.extensions.spiderstate.SpiderState": 0,        # 爬虫状态
    "scrapy.extensions.throttle.AutoThrottle": 0,          # 自动限速
}
```

### 5.2 中间件系统

#### 5.2.1 下载器中间件

```python
# default_settings.py:283-300
DOWNLOADER_MIDDLEWARES_BASE = {
    # 按优先级排序（数字越小越先执行）
    "scrapy.downloadermiddlewares.offsite.OffsiteMiddleware": 50,
    "scrapy.downloadermiddlewares.robotstxt.RobotsTxtMiddleware": 100,
    "scrapy.downloadermiddlewares.httpauth.HttpAuthMiddleware": 300,
    "scrapy.downloadermiddlewares.downloadtimeout.DownloadTimeoutMiddleware": 350,
    "scrapy.downloadermiddlewares.defaultheaders.DefaultHeadersMiddleware": 400,
    "scrapy.downloadermiddlewares.useragent.UserAgentMiddleware": 500,
    "scrapy.downloadermiddlewares.retry.RetryMiddleware": 550,
    "scrapy.downloadermiddlewares.ajaxcrawl.AjaxCrawlMiddleware": 560,
    "scrapy.downloadermiddlewares.redirect.MetaRefreshMiddleware": 580,
    "scrapy.downloadermiddlewares.httpcompression.HttpCompressionMiddleware": 590,
    "scrapy.downloadermiddlewares.redirect.RedirectMiddleware": 600,
    "scrapy.downloadermiddlewares.cookies.CookiesMiddleware": 700,
    "scrapy.downloadermiddlewares.httpproxy.HttpProxyMiddleware": 750,
    "scrapy.downloadermiddlewares.stats.DownloaderStats": 850,
    "scrapy.downloadermiddlewares.httpcache.HttpCacheMiddleware": 900,
}
```

#### 5.2.2 爬虫中间件

```python
# default_settings.py:504-511
SPIDER_MIDDLEWARES_BASE = {
    "scrapy.spidermiddlewares.start.StartSpiderMiddleware": 25,
    "scrapy.spidermiddlewares.httperror.HttpErrorMiddleware": 50,
    "scrapy.spidermiddlewares.referer.RefererMiddleware": 700,
    "scrapy.spidermiddlewares.urllength.UrlLengthMiddleware": 800,
    "scrapy.spidermiddlewares.depth.DepthMiddleware": 900,
}
```

### 5.3 信号系统

Crawler 通过 `SignalManager` 实现事件驱动：

**核心信号列表**（`scrapy/signals.py`）：

| 信号 | 触发时机 | 用途 |
|------|----------|------|
| `engine_started` | 引擎启动时 | 初始化扩展 |
| `engine_stopped` | 引擎停止时 | 清理资源 |
| `spider_opened` | Spider 打开时 | 组件初始化 |
| `spider_closed` | Spider 关闭时 | 资源清理、数据持久化 |
| `spider_idle` | Spider 空闲时 | 检查是否关闭爬虫 |
| `request_scheduled` | 请求入队时 | 监控、统计 |
| `request_dropped` | 请求被丢弃时 | 统计、日志 |
| `response_received` | 响应收到时 | 统计、缓存检查 |
| `item_scraped` | Item 产出时 | 数据处理 |
| `item_dropped` | Item 被丢弃时 | 日志、统计 |

### 5.4 关键配置项汇总

#### 并发控制

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `CONCURRENT_REQUESTS` | 16 | 全局最大并发请求数 |
| `CONCURRENT_REQUESTS_PER_DOMAIN` | 8 | 单域名最大并发 |
| `CONCURRENT_REQUESTS_PER_IP` | 0 | 单 IP 最大并发（0=禁用） |
| `CONCURRENT_ITEMS` | 100 | 并发处理的 Item 数 |

#### 下载控制

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `DOWNLOAD_DELAY` | 0 | 下载延迟（秒） |
| `RANDOMIZE_DOWNLOAD_DELAY` | True | 随机化延迟（0.5-1.5 倍） |
| `DOWNLOAD_TIMEOUT` | 180 | 下载超时（秒） |
| `RETRY_TIMES` | 2 | 重试次数 |

#### 队列配置

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `SCHEDULER_PRIORITY_QUEUE` | `DownloaderAwarePriorityQueue` | 优先级队列实现 |
| `SCHEDULER_MEMORY_QUEUE` | `LifoMemoryQueue` | 内存队列（LIFO=DFS） |
| `SCHEDULER_DISK_QUEUE` | `PickleLifoDiskQueue` | 磁盘队列 |
| `JOBDIR` | `None` | 持久化目录（启用断点续传） |

#### 深度/优先级

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `DEPTH_LIMIT` | 0 | 爬取深度限制（0=无限制） |
| `DEPTH_PRIORITY` | 0 | 深度优先级调整 |

---

## 附录：完整调用链路

### A.1 单次请求处理链路

```
Crawler.crawl_async()
    │
    ├──▶ ExecutionEngine.open_spider_async()
    │       ├──▶ Scheduler.open()          # 初始化内存/磁盘队列
    │       ├──▶ Scraper.open_spider_async()
    │       └──▶ 发送 spider_opened 信号
    │
    └──▶ ExecutionEngine.start_async()
            │
            └──▶ _start_request_processing()
                    │
                    ├──▶ _process_start_next()      # 处理 start_requests
                    │       └──▶ crawl(request)
                    │               └──▶ _schedule_request()
                    │                       └──▶ Scheduler.enqueue_request()
                    │
                    └──▶ _start_scheduled_requests()  # 主循环
                            │
                            └──▶ _start_scheduled_request()
                                    │
                                    ├──▶ Scheduler.next_request()    # 取请求
                                    │
                                    ├──▶ _download(request)
                                    │       └──▶ Downloader.fetch()
                                    │               ├──▶ 下载器中间件处理
                                    │               └──▶ 实际 HTTP 请求
                                    │
                                    └──▶ _handle_downloader_output()
                                            │
                                            ├──▶ Scraper.enqueue_scrape()
                                            │       │
                                            │       ├──▶ Spider 回调解析
                                            │       │
                                            │       ├──▶ 产出 Item → Item Pipeline
                                            │       │
                                            │       └──▶ 产出 Request → crawl()
                                            │               └──▶ 重新入队调度器
                                            │
                                            └──▶ 循环继续...
```

### A.2 组件交互图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              组件交互时序                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Engine          Scheduler      Downloader       Scraper       Spider      │
│    │                │              │               │              │         │
│    │  next_request()│              │               │              │         │
│    │───────────────▶│              │               │              │         │
│    │                │              │               │              │         │
│    │   Request      │              │               │              │         │
│    │◀───────────────│              │               │              │         │
│    │                │              │               │              │         │
│    │   fetch()      │              │               │              │         │
│    │──────────────────────────────▶│               │              │         │
│    │                │              │               │              │         │
│    │                │              │  HTTP 请求    │              │         │
│    │                │              │──────────────▶│              │         │
│    │                │              │               │              │         │
│    │                │              │   Response    │              │         │
│    │                │              │◀──────────────│              │         │
│    │                │              │               │              │         │
│    │   Response     │              │               │              │         │
│    │◀──────────────────────────────│               │              │         │
│    │                │              │               │              │         │
│    │  enqueue_scrape()             │               │              │         │
│    │──────────────────────────────────────────────▶│              │         │
│    │                │              │               │              │         │
│    │                │              │               │  callback()  │         │
│    │                │              │               │─────────────▶│         │
│    │                │              │               │              │         │
│    │                │              │               │   Item/Request          │
│    │                │              │               │◀─────────────│         │
│    │                │              │               │              │         │
│    │◀───────────────│              │               │              │         │
│    │  crawl(new_rq) │              │               │              │         │
│    │───────────────▶│              │               │              │         │
│    │                │              │               │              │         │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 总结

Scrapy 的核心调度架构体现了优秀的设计思想：

1. **分层架构**：Crawler → Engine → Scheduler/Downloader/Scraper，职责清晰
2. **事件驱动**：通过信号系统实现组件解耦
3. **双队列设计**：内存队列保证速度，磁盘队列支持持久化和断点续传
4. **优先级调度**：基于请求优先级和下载器负载的智能调度
5. **背压控制**：通过 needs_backout 机制防止系统过载
6. **可扩展性**：中间件、扩展、管道的插件化设计

这种架构使得 Scrapy 既能高效处理大规模爬取任务，又能灵活适应各种定制需求。
