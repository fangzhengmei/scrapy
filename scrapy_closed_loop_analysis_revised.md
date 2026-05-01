# Scrapy 请求调度闭环分析报告（修订版）

## 修订说明

本报告对上一版闭环分析中的**调度触发点清单**进行了源码级核对和纠偏，确保时序图与文字说明与源码真实行为完全一致。

### 上一版的主要偏差

| 上一版说法 | 实际源码行为 |
|-----------|-------------|
| "Spider 打开时" 在 `open_spider_async` 中触发 | 实际在 `_start_request_processing` 方法中触发 |
| "Item 处理完成时" 在 `enqueue_scrape()` 的 `finally` 块中触发 | **错误** - `enqueue_scrape` 调用的是 `Scraper._scrape_next()`，不是 `_Slot.nextcall.schedule()` |

---

## 1. 核心概念澄清

在深入分析之前，需要明确两个容易混淆的概念：

### 1.1 `_Slot.nextcall` vs `Scraper._scrape_next()`

| 概念 | 所属类 | 功能 | 触发对象 |
|------|--------|------|---------|
| **`_Slot.nextcall`** | `ExecutionEngine._Slot` | 包装 `_start_scheduled_requests` 的 `CallLaterOnce`，用于**从调度器取出请求并发送到下载器** | `Engine` 层调度 |
| **`Scraper._scrape_next()`** | `Scraper` | 处理下一个已入队的响应，用于**Spider 回调和 Item 处理** | `Scraper` 层内部 |

**关键区别**：
- `_Slot.nextcall.schedule()` 触发的是**请求调度**（从调度器取请求）
- `Scraper._scrape_next()` 触发的是**响应处理**（处理已下载的响应）

### 1.2 `CallLaterOnce` 的工作机制

位置: `scrapy/utils/reactor.py:47-90`

```python
class CallLaterOnce(Generic[_T]):
    def __init__(self, func, *a, **kw):
        self._func = func
        self._a = a
        self._kw = kw
        self._call = None  # 关键：防止重复调度的标记

    def schedule(self, delay: float = 0) -> None:
        if self._call is None:  # 只有未调度时才调度
            self._call = call_later(delay, self)

    def __call__(self) -> _T:
        self._call = None  # 执行时重置，允许下次调度
        return self._func(*self._a, **self._kw)
```

**核心特性**：
- **幂等性**：多次调用 `schedule()` 只有第一次生效
- **延迟执行**：在下一个 reactor 循环中执行 `_func`
- **`_func` 是什么**：对于 `_Slot.nextcall`，`_func` 是 `_start_scheduled_requests`

---

## 2. `nextcall.schedule()` 真实触发位置（源码核对）

经过对 `scrapy/core/engine.py` 的逐条核对，以下是 `_Slot.nextcall.schedule()` 的**所有真实触发位置**：

### 2.1 触发点清单

| # | 触发位置 | 代码行号 | 触发时机 | 调用方式 |
|---|---------|---------|---------|---------|
| 1 | `_start_request_processing` 开始 | 306 | 请求处理循环初始化时 | 直接调用 |
| 2 | `_start_request_processing` 循环中 | 314 | 处理 `Spider.start()` 输出时 | 直接调用 |
| 3 | `_start_scheduled_request` 回调 | 387 | 请求处理完成后（Deferred 回调） | `d2.addBoth(lambda _: slot.nextcall.schedule())` |
| 4 | `_download` 的 `finally` 块 | 517 | 下载完成后（无论成功失败） | 直接调用 |
| 5 | `crawl` 方法 | 437 | 新请求入队后 | 直接调用 |
| 6 | `_Slot.heartbeat` 定时 | 76-78 | 每 5 秒（安全网） | `create_looping_call(nextcall.schedule)` |

### 2.2 各触发点源码详解

#### 触发点 1 & 2：`_start_request_processing` 方法

位置: `scrapy/core/engine.py:298-327`

```python
async def _start_request_processing(self) -> None:
    try:
        assert self._slot is not None
        
        # ====== 触发点 1: 循环开始时 ======
        self._slot.nextcall.schedule()  # 第 306 行
        
        # 启动心跳定时
        self._slot.heartbeat.start(self._SLOT_HEARTBEAT_INTERVAL)

        while self._start and self.spider and self.running:
            await self._process_start_next()  # 处理 Spider.start() 的输出
            
            if not self.needs_backout():
                # ====== 触发点 2: 循环中 ======
                self._slot.nextcall.schedule()  # 第 314 行
                await self._slot.nextcall.wait()
    except:
        # ... 异常处理
```

**说明**：
- 触发点 1：在请求处理循环**开始时**触发一次，启动整个调度流程
- 触发点 2：在处理 `Spider.start()` 输出的**循环中**触发，确保 start requests 能被及时调度

#### 触发点 3：`_start_scheduled_request` 中的 Deferred 回调

位置: `scrapy/core/engine.py:355-395`

```python
def _start_scheduled_request(self) -> bool:
    assert self._slot is not None
    assert self.spider is not None

    # 1. 从调度器取出请求
    request = self._slot.scheduler.next_request()
    if request is None:
        self.signals.send_catch_log(signals.scheduler_empty)
        return False

    # 2. 下载请求
    d: Deferred[Response | Request] = self._download(request)
    d.addBoth(self._handle_downloader_output, request)
    
    # 3. 定义清理回调
    def _remove_request(_: Any) -> None:
        assert self._slot
        self._slot.remove_request(request)
    
    d2: Deferred[None] = d.addBoth(_remove_request)
    
    # ====== 触发点 3: 请求处理完成后的回调 ======
    slot = self._slot
    d2.addBoth(lambda _: slot.nextcall.schedule())  # 第 387 行
    
    return True
```

**说明**：
- 这是**主闭环的核心触发点**
- 当一个请求的整个处理流程（下载→响应处理→Item 处理）完成后，通过 `d2.addBoth()` 回调触发下一次调度
- 无论成功还是失败，`addBoth` 都会执行

#### 触发点 4：`_download` 的 `finally` 块

位置: `scrapy/core/engine.py:483-517`

```python
@inlineCallbacks
def _download(
    self, request: Request
) -> Generator[Deferred[Any], Any, Response | Request]:
    assert self._slot is not None
    assert self.spider is not None

    self._slot.add_request(request)  # 标记为进行中
    try:
        # 下载请求
        if self._downloader_fetch_needs_spider:
            result = yield self.downloader.fetch(request, self.spider)
        else:
            result = yield self.downloader.fetch(request)
        
        if isinstance(result, Response):
            # ... 日志记录
            self.signals.send_catch_log(
                signal=signals.response_received,  # CoreStats 订阅此信号
                response=result,
                request=result.request,
                spider=self.spider,
            )
        return result
    finally:
        # ====== 触发点 4: 下载完成后 ======
        self._slot.nextcall.schedule()  # 第 517 行
```

**说明**：
- 在 `finally` 块中，确保**无论下载成功还是失败**都会触发
- 这与触发点 3 形成"双重保险"

#### 触发点 5：`crawl` 方法

位置: `scrapy/core/engine.py:432-437`

```python
def crawl(self, request: Request) -> None:
    """Inject the request into the spider <-> downloader pipeline"""
    if self.spider is None:
        raise RuntimeError(f"No open spider to crawl: {request}")
    self._schedule_request(request)  # 入队到调度器
    
    # ====== 触发点 5: 新请求入队后 ======
    self._slot.nextcall.schedule()  # 第 437 行
```

**说明**：
- 当 Spider 回调产生新的 Request 时（如提取的链接），通过 `crawl()` 方法入队
- 入队后立即触发调度，确保新请求能被及时处理

#### 触发点 6：`_Slot.heartbeat` 定时触发

位置: `scrapy/core/engine.py:64-78`

```python
class _Slot:
    def __init__(
        self,
        close_if_idle: bool,
        nextcall: CallLaterOnce[None],
        scheduler: BaseScheduler,
    ) -> None:
        # ...
        self.nextcall: CallLaterOnce[None] = nextcall
        
        # ====== 触发点 6: 心跳定时 ======
        self.heartbeat: AsyncioLoopingCall | LoopingCall = create_looping_call(
            nextcall.schedule  # 第 77 行
        )
```

位置: `scrapy/core/engine.py:102`

```python
class ExecutionEngine:
    _SLOT_HEARTBEAT_INTERVAL: float = 5.0  # 每 5 秒
```

**说明**：
- 这是一个**安全网机制**
- 每 5 秒触发一次，防止因某些异常情况导致调度器停止
- 由于 `CallLaterOnce` 的幂等性，如果已有调度在进行，不会重复调度

---

## 3. 修正后的闭环时序图

### 3.1 主闭环（单次请求生命周期）

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                    修正后的请求调度闭环（单次请求生命周期）                            │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │ 【起点】_start_scheduled_request() 被调用                                       │  │
│  │  (由 nextcall.schedule() 延迟触发，实际执行 _func = _start_scheduled_requests) │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                      │                                               │
│                                      ▼                                               │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │ 阶段 1: 请求出队                                                                │  │
│  │ ┌──────────────────────────────────────────────────────────────────────────┐ │  │
│  │ │ scheduler.next_request()                                                   │ │  │
│  │ │   └── 从内存/磁盘队列取出请求                                                │ │  │
│  │ │   └── 统计更新: scheduler/dequeued, scheduler/dequeued/memory/disk        │ │  │
│  │ └──────────────────────────────────────────────────────────────────────────┘ │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                      │                                               │
│                                      ▼                                               │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │ 阶段 2: 下载请求                                                                │  │
│  │ ┌──────────────────────────────────────────────────────────────────────────┐ │  │
│  │ │ _download(request)                                                         │ │  │
│  │ │   ├── self._slot.add_request(request)  # 标记为进行中                      │ │  │
│  │ │   ├── downloader.fetch(request)  # 实际下载                                │ │  │
│  │ │   │   └── DownloaderStats.process_request()  # 统计 request_count 等       │ │  │
│  │ │   │                                                               │ │  │
│  │ │   ├── [下载完成后]                                                  │ │  │
│  │ │   │   ├── Downloader._download() 发射 response_downloaded 信号    │ │  │
│  │ │   │   │   └── AutoThrottle._response_downloaded()  # 调整延迟     │ │  │
│  │ │   │   │                                                             │ │  │
│  │ │   │   ├── Engine._download() 发射 response_received 信号          │ │  │
│  │ │   │   │   └── CoreStats.response_received()  # response_received_count++││  │
│  │ │   │   │                                                             │ │  │
│  │ │   │   └── DownloaderStats.process_response/exception()              │ │  │
│  │ │   │       └── 统计 response_count 或 exception_count                 │ │  │
│  │ │   │                                                               │ │  │
│  │ │   └── [finally 块] 触发点 4: self._slot.nextcall.schedule()       │ │  │
│  │ │       (无论下载成功失败都会触发)                                      │ │  │
│  │ └──────────────────────────────────────────────────────────────────────────┘ │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                      │                                               │
│                                      ▼                                               │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │ 阶段 3: 响应处理                                                                │  │
│  │ ┌──────────────────────────────────────────────────────────────────────────┐ │  │
│  │ │ _handle_downloader_output(result, request)                                │ │  │
│  │ │   ├── 如果 result 是 Request（如重定向）:                                   │ │  │
│  │ │   │   └── self.crawl(result)  # 新请求入队                                 │ │  │
│  │ │   │       └── 触发点 5: self._slot.nextcall.schedule()                    │ │  │
│  │ │   │                                                               │ │  │
│  │ │   └── 如果 result 是 Response/Failure:                                    │ │  │
│  │ │       └── yield self.scraper.enqueue_scrape(result, request)              │ │  │
│  │ │           ├── Spider 回调执行（可能产生新的 Request/Item）                  │ │  │
│  │ │           │                                                         │ │  │
│  │ │           ├── [如果产生新 Request]                                         │ │  │
│  │ │           │   └── 回调中调用 engine.crawl(request)                        │ │  │
│  │ │           │       └── 触发点 5 被触发                                      │ │  │
│  │ │           │                                                         │ │  │
│  │ │           ├── [如果产生 Item]                                              │ │  │
│  │ │           │   ├── Item Pipeline 处理                                       │ │  │
│  │ │           │   ├── 成功: 发射 item_scraped 信号                            │ │  │
│  │ │           │   │   └── CoreStats.item_scraped()  # item_scraped_count++   │ │  │
│  │ │           │   └── 失败: 发射 item_dropped 信号                             │ │  │
│  │ │           │       └── CoreStats.item_dropped()  # 统计丢弃数              │ │  │
│  │ │           │                                                         │ │  │
│  │ │           └── ⚠️ 注意: enqueue_scrape 的 finally 块调用的是               │ │  │
│  │ │               Scraper._scrape_next()，不是 _Slot.nextcall.schedule()       │ │  │
│  │ │               （这是 Scraper 内部的响应处理循环，不是 Engine 的请求调度）   │ │  │
│  │ └──────────────────────────────────────────────────────────────────────────┘ │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                      │                                               │
│                                      ▼                                               │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │ 阶段 4: 请求处理完成（Deferred 回调链）                                         │  │
│  │ ┌──────────────────────────────────────────────────────────────────────────┐ │  │
│  │ │ _start_scheduled_request() 中的回调链                                      │ │  │
│  │ │                                                                              │ │  │
│  │ │ d = _download(request)                                                      │ │  │
│  │ │ d.addBoth(_handle_downloader_output, request)  # 阶段 3                    │ │  │
│  │ │ d2 = d.addBoth(_remove_request)  # 从 inprogress 移除                      │ │  │
│  │ │                                                                              │ │  │
│  │ │ ====== 触发点 3: 主闭环核心 ======                                          │ │  │
│  │ │ d2.addBoth(lambda _: slot.nextcall.schedule())  # 第 387 行               │ │  │
│  │ │                                                                              │ │  │
│  │ │ 说明: 当整个处理流程（下载→响应处理→Item 处理）完成后，                      │ │  │
│  │ │       触发下一次调度，形成闭环                                                │ │  │
│  │ └──────────────────────────────────────────────────────────────────────────┘ │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                      │                                               │
│                                      ▼                                               │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │ 【终点】回到起点: _start_scheduled_requests() 被延迟调度                       │  │
│  │  (等待下一个 reactor 循环执行)                                                  │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 启动阶段的触发顺序

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         启动阶段的触发顺序                                             │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Crawler.crawl_async()                                                              │
│       │                                                                              │
│       ▼                                                                              │
│  engine.open_spider_async()                                                         │
│       │                                                                              │
│       ├── 创建 _Slot:                                                               │
│       │   nextcall = CallLaterOnce(self._start_scheduled_requests)                │
│       │   heartbeat = create_looping_call(nextcall.schedule)  # 触发点 6 准备    │
│       │                                                                              │
│       └── 发射 spider_opened 信号                                                   │
│           ├── AutoThrottle._spider_opened()  # 初始化 delay                       │
│           ├── CoreStats.spider_opened()  # 记录 start_time                        │
│           └── LogStats.spider_opened()  # 启动定时任务                            │
│                                                                                      │
│       ▼                                                                              │
│  engine.start_async()                                                                │
│       │                                                                              │
│       └── _start_request_processing()                                                │
│            │                                                                         │
│            ├── 触发点 1: self._slot.nextcall.schedule()  # 第 306 行             │
│            │       (首次调度，启动整个爬取流程)                                      │
│            │                                                                         │
│            └── 启动 heartbeat:                                                       │
│                self._slot.heartbeat.start(self._SLOT_HEARTBEAT_INTERVAL)           │
│                (触发点 6 开始定时执行，每 5 秒一次)                                  │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 4. 三个扩展在闭环中的协作（修正后）

### 4.1 协作总览

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                    三个扩展在闭环中的协作时序（修正后）                                │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  时间轴（单次请求闭环） ───────────────────────────────────────────────────────────▶│
│                                                                                      │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │ 1. 请求出队 (scheduler.next_request)                                          │  │
│  │    └── 统计更新: scheduler/dequeued/* (调度器内部，非扩展)                    │  │
│  │    └── 【扩展参与】: 无                                                        │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                      │                                               │
│                                      ▼                                               │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │ 2. 请求发送 (downloader.fetch)                                                │  │
│  │    └── DownloaderStats.process_request()                                       │  │
│  │        └── 统计: downloader/request_count, request_bytes, etc.               │  │
│  │    └── 【扩展参与】: DownloaderStats（下载器中间件，非 Extension）             │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                      │                                               │
│                                      ▼                                               │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │ 3. 响应下载完成 (Downloader._download)                                         │  │
│  │    └── 信号: response_downloaded                                               │  │
│  │        └── AutoThrottle._response_downloaded()                                │  │
│  │            ├── 读取 request.meta['download_latency']                          │  │
│  │            ├── 获取 Slot: crawler.engine.downloader.slots[key]               │  │
│  │            └── 调用 _adjust_delay() 修改 slot.delay                           │  │
│  │    └── 【扩展参与】: AutoThrottle ✅                                           │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                      │                                               │
│                                      ▼                                               │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │ 4. 响应接收 (Engine._download)                                                │  │
│  │    └── 信号: response_received                                                │  │
│  │        └── CoreStats.response_received()                                      │  │
│  │            └── 统计: response_received_count += 1                             │  │
│  │    └── DownloaderStats.process_response() 或 process_exception()              │  │
│  │        └── 统计: response_count 或 exception_count                            │  │
│  │    └── 【扩展参与】: CoreStats ✅                                               │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                      │                                               │
│                                      ▼                                               │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │ 5. Item 处理完成 (Scraper.start_itemproc_async)                              │  │
│  │    ├── 成功: 信号 item_scraped                                                │  │
│  │    │   └── CoreStats.item_scraped()                                           │  │
│  │    │       └── 统计: item_scraped_count += 1                                  │  │
│  │    └── 失败: 信号 item_dropped                                                │  │
│  │        └── CoreStats.item_dropped()                                           │  │
│  │            └── 统计: item_dropped_count += 1 + 按原因分类                     │  │
│  │    └── 【扩展参与】: CoreStats ✅                                               │  │
│  │                                                                                 │  │
│  │    ⚠️ 注意: 此处不会触发 _Slot.nextcall.schedule()                              │  │
│  │       enqueue_scrape 的 finally 块调用的是 Scraper._scrape_next()             │  │
│  │       （这是 Scraper 内部的响应处理循环，不是 Engine 的请求调度）              │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                      │                                               │
│                                      ▼                                               │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │ 6. 新请求入队 (如果 Spider 回调产生新 Request)                                 │  │
│  │    └── engine.crawl(request)                                                   │  │
│  │        ├── _schedule_request(request)  # 入队调度器                           │  │
│  │        │   ├── 信号: request_scheduled                                        │  │
│  │        │   └── 统计: scheduler/enqueued/*                                     │  │
│  │        └── 触发点 5: self._slot.nextcall.schedule()  # 第 437 行            │  │
│  │    └── 【扩展参与】: 无直接参与，但新请求会进入下一个闭环                      │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                      │                                               │
│                                      ▼                                               │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │ 7. Deferred 回调链完成 (触发点 3)                                             │  │
│  │    └── d2.addBoth(lambda _: slot.nextcall.schedule())  # 第 387 行          │  │
│  │        (这是主闭环的核心触发点，确保请求处理完成后调度下一个)                  │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                      │
│  ═══════════════════════════════════════════════════════════════════════════════  │
│                                                                                      │
│  【独立于单次闭环】: LogStats 定时任务                                               │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │ 每 LOGSTATS_INTERVAL 秒（默认 60 秒）触发一次                                 │  │
│  │                                                                                 │  │
│  │ LogStats.calculate_stats()                                                     │  │
│  │   ├── 从 StatsCollector 读取:                                                  │  │
│  │   │   ├── response_received_count → pages                                      │  │
│  │   │   └── item_scraped_count → items                                          │  │
│  │   ├── 计算速率:                                                                │  │
│  │   │   ├── prate = (pages - pagesprev) * multiplier (pages/min)               │  │
│  │   │   └── irate = (items - itemsprev) * multiplier (items/min)               │  │
│  │   └── 输出日志                                                                  │  │
│  │                                                                                 │  │
│  │ 【扩展参与】: LogStats ✅ (消费者角色)                                          │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 各扩展的协作细节

#### AutoThrottle：动态限速调整

**订阅的信号**：`response_downloaded`

**触发位置**：`scrapy/core/downloader/__init__.py:229-234`（Downloader 的 `_download` 方法）

**工作流程**：
```python
# Downloader._download 中发射信号
self.signals.send_catch_log(
    signal=signals.response_downloaded,
    response=response,
    request=request,
    spider=self.crawler.spider,
)

# AutoThrottle 回调
def _response_downloaded(self, response, request, spider):
    key, slot = self._get_slot(request, spider)
    latency = request.meta.get("download_latency")
    
    if latency is None or slot is None:
        return
    
    # 调整延迟
    self._adjust_delay(slot, latency, response)
```

**与下载器的交互**：
- AutoThrottle **不直接触发调度**，它只修改 `Slot.delay`
- 下载器在 `_process_queue` 中使用 `slot.delay` 来决定是否延迟发送请求
- 这是一种**被动控制**：通过修改延迟值影响后续请求的发送时机

#### CoreStats：核心统计收集

**订阅的信号**：
- `response_received` → 统计响应数
- `item_scraped` → 统计成功抓取数
- `item_dropped` → 统计丢弃数

**触发位置**：
- `response_received`: `scrapy/core/engine.py:509-514`（Engine 的 `_download` 方法）
- `item_scraped`/`item_dropped`: `scrapy/core/scraper.py:513-547`（Scraper 的 `start_itemproc_async` 方法）

**工作流程**：
```python
# Engine._download 中发射 response_received
self.signals.send_catch_log(
    signal=signals.response_received,
    response=result,
    request=result.request,
    spider=self.spider,
)

# CoreStats 回调
def response_received(self, spider):
    self.stats.inc_value("response_received_count")

# Scraper.start_itemproc_async 中发射 item_scraped
await self.signals.send_catch_log_async(
    signal=signals.item_scraped,
    item=output,
    response=response,
    spider=self.crawler.spider,
)

# CoreStats 回调
def item_scraped(self, item, spider):
    self.stats.inc_value("item_scraped_count")
```

#### LogStats：统计数据消费与日志输出

**工作模式**：定时任务（独立于单次请求闭环）

**触发机制**：
```python
# LogStats.spider_opened 中启动定时任务
def spider_opened(self, spider):
    self.pagesprev = 0
    self.itemsprev = 0
    
    # 创建定时循环，每 interval 秒调用 self.log
    self.task = create_looping_call(self.log, spider)
    self.task.start(self.interval)

# 定时执行的方法
def log(self, spider):
    self.calculate_stats()
    
    # 输出日志
    logger.info(
        "Crawled %(pages)d pages (at %(pagerate)d pages/min), "
        "scraped %(items)d items (at %(itemrate)d items/min)",
        log_args, extra={"spider": spider}
    )

def calculate_stats(self):
    # 从 StatsCollector 读取数据
    self.items = self.stats.get_value("item_scraped_count", 0)
    self.pages = self.stats.get_value("response_received_count", 0)
    
    # 计算速率（与上次的差值）
    self.irate = (self.items - self.itemsprev) * self.multiplier
    self.prate = (self.pages - self.pagesprev) * self.multiplier
    
    # 更新上次值
    self.pagesprev, self.itemsprev = self.pages, self.items
```

**关键特点**：
- LogStats **不订阅任何请求处理相关的信号**
- 它通过 `create_looping_call` 创建定时任务，定期从 `StatsCollector` 读取数据
- 这是一种**生产者-消费者**模式：CoreStats 等是生产者，LogStats 是消费者

---

## 5. 关键纠偏总结

### 5.1 上一版的错误点

| 错误说法 | 正确事实 | 源码证据 |
|---------|---------|---------|
| "Item 处理完成时" 在 `enqueue_scrape()` 的 `finally` 块中触发 `nextcall.schedule()` | `enqueue_scrape` 的 `finally` 块调用的是 `Scraper._scrape_next()`，**不是** `_Slot.nextcall.schedule()` | `scrapy/core/scraper.py:237-240` |
| "Spider 打开时" 在 `open_spider_async` 中触发 | 实际在 `_start_request_processing` 方法中触发（第 306 行） | `scrapy/core/engine.py:306` |

### 5.2 `_scrape_next()` vs `nextcall.schedule()` 的区别

| 特性 | `Scraper._scrape_next()` | `_Slot.nextcall.schedule()` |
|------|--------------------------|-----------------------------|
| 所属层 | Scraper 层（响应处理） | Engine 层（请求调度） |
| 功能 | 处理下一个已入队的**响应** | 调度下一个待发送的**请求** |
| 调用时机 | `enqueue_scrape` 的 `finally` 块 | 见触发点 1-6 |
| 与闭环的关系 | **不触发**请求调度闭环 | **是**闭环的核心触发机制 |

**源码证据**：

```python
# Scraper.enqueue_scrape 的 finally 块
# scrapy/core/scraper.py:237-240
finally:
    self.slot.finish_response(result, request)
    self._check_if_closing()
    self._scrape_next()  # 这是 Scraper 内部方法！

# Scraper._scrape_next 的实现
# scrapy/core/scraper.py:242-246
def _scrape_next(self) -> None:
    assert self.slot is not None
    while self.slot.queue:
        result, request, queue_dfd = self.slot.next_response_request_deferred()
        _schedule_coro(self._wait_for_processing(result, request, queue_dfd))
```

可以看到，`_scrape_next()` 只是处理 Scraper 内部的响应队列，**完全不涉及** `_Slot.nextcall` 或请求调度。

### 5.3 真实的闭环触发机制

主闭环的核心触发点是 **触发点 3**（第 387 行）：

```python
# scrapy/core/engine.py:387
d2.addBoth(lambda _: slot.nextcall.schedule())
```

这个回调在以下时机执行：
1. `_download(request)` 完成（无论成功失败）
2. `_handle_downloader_output` 完成（响应处理完成）
3. `_remove_request` 完成（从 `inprogress` 移除）

只有当这整个链条完成后，才会触发下一次调度。

其他触发点（1、2、4、5、6）是辅助机制：
- 触发点 1、2：启动阶段和处理 `start()` 输出时
- 触发点 4：下载完成后的 `finally` 块（双重保险）
- 触发点 5：新请求入队后（及时调度新请求）
- 触发点 6：心跳定时（安全网）

---

## 6. 完整触发点清单（最终版）

| # | 触发位置 | 代码行号 | 触发场景 | 是主闭环吗 |
|---|---------|---------|---------|-----------|
| 1 | `_start_request_processing` 开始 | 306 | 请求处理循环初始化 | 否（启动阶段） |
| 2 | `_start_request_processing` 循环中 | 314 | 处理 `Spider.start()` 输出 | 否（start requests 专用） |
| 3 | `_start_scheduled_request` 回调 | 387 | 请求处理完成后 | **是（主闭环核心）** |
| 4 | `_download` 的 `finally` 块 | 517 | 下载完成后 | 否（双重保险） |
| 5 | `crawl` 方法 | 437 | 新请求入队后 | 否（新请求专用） |
| 6 | `_Slot.heartbeat` 定时 | 76-78 | 每 5 秒 | 否（安全网） |

---

**报告修订时间**: 2026-05-01  
**源码核对依据**: `g:\fangzheng\solo-dogfeeding\code\17679-scrapy\scrapy\core\engine.py`
