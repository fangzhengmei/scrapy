# Scrapy 配置系统分析报告

## 1. 配置优先级槽位机制

### 1.1 优先级定义

Scrapy 通过 `SETTINGS_PRIORITIES` 字典定义了不同配置来源的优先级数值：

```python
SETTINGS_PRIORITIES: dict[str, int] = {
    "default": 0,    # 最低优先级 - 内置默认值
    "command": 10,   # 命令级配置
    "addon": 15,     # 插件配置
    "project": 20,   # 项目配置 (settings.py)
    "spider": 30,    # 爬虫自定义配置
    "cmdline": 40,   # 命令行参数 (最高优先级)
}
```

**关键代码位置**: `scrapy/settings/__init__.py:35-42`

### 1.2 优先级辅助函数

`get_settings_priority` 函数负责将字符串优先级名称转换为对应的数值：

```python
def get_settings_priority(priority: int | str) -> int:
    if isinstance(priority, str):
        return SETTINGS_PRIORITIES[priority]
    return priority
```

**关键代码位置**: `scrapy/settings/__init__.py:45-53`

### 1.3 配置属性存储

配置通过 `SettingsAttribute` 类存储，包含值和优先级信息：

```python
class SettingsAttribute:
    def __init__(self, value: Any, priority: int):
        self.value: Any = value
        self.priority: int
        if isinstance(self.value, BaseSettings):
            self.priority = max(self.value.maxpriority(), priority)
        else:
            self.priority = priority

    def set(self, value: Any, priority: int) -> None:
        """只有当新优先级 >= 当前优先级时才更新值"""
        if priority >= self.priority:
            if isinstance(self.value, BaseSettings):
                value = BaseSettings(value, priority=priority)
            self.value = value
            self.priority = priority
```

**关键代码位置**: `scrapy/settings/__init__.py:56-80`

### 1.4 优先级合并策略

`BaseSettings.set` 方法实现了优先级检查的核心逻辑：

```python
def set(
    self, name: _SettingsKey, value: Any, priority: int | str = "project"
) -> None:
    self._assert_mutability()
    priority = get_settings_priority(priority)
    if name not in self:
        # 新配置直接添加
        if isinstance(value, SettingsAttribute):
            self.attributes[name] = value
        else:
            self.attributes[name] = SettingsAttribute(value, priority)
    else:
        # 已有配置，检查优先级
        self.attributes[name].set(value, priority)
```

**关键代码位置**: `scrapy/settings/__init__.py:459-487`

**核心规则**：
- 新配置若不存在于 `attributes` 中，直接添加
- 已有配置只有在 **新优先级 >= 当前优先级** 时才会更新
- 这保证了高优先级来源的配置不会被低优先级来源覆盖

## 2. 命令内置配置层（command，优先级10）

### 2.1 命令配置声明方式

每个 `ScrapyCommand` 子类通过 `default_settings` 类属性声明该命令的专属默认配置：

```python
class ScrapyCommand(ABC):
    # default settings to be used for this command instead of global defaults
    default_settings: ClassVar[dict[str, Any]] = {}
```

**关键代码位置**: `scrapy/commands/__init__.py:27-33`

### 2.2 实际命令配置示例

多个内置命令使用 `default_settings` 声明专属配置：

**bench 命令** - 性能测试命令：
```python
class Command(ScrapyCommand):
    default_settings: ClassVar[dict[str, Any]] = {
        "LOG_LEVEL": "INFO",
        "LOGSTATS_INTERVAL": 1,
        "CLOSESPIDER_TIMEOUT": 10,
    }
```
**关键代码位置**: `scrapy/commands/bench.py:20-25`

**shell 命令** - 交互式控制台：
```python
class Command(ScrapyCommand):
    default_settings: ClassVar[dict[str, Any]] = {
        "DUPEFILTER_CLASS": "scrapy.dupefilters.BaseDupeFilter",
        "KEEP_ALIVE": True,
        "LOGSTATS_INTERVAL": 0,
    }
```
**关键代码位置**: `scrapy/commands/shell.py:27-32`

**禁用日志的命令** - 多个命令默认禁用日志输出：
```python
# check, edit, genspider, list, settings, startproject, version 等命令
default_settings: ClassVar[dict[str, Any]] = {"LOG_ENABLED": False}
```

### 2.3 配置注入触发入口

命令内置配置的注入发生在 `scrapy.cmdline.execute()` 函数中：

```python
def execute(argv: list[str] | None = None, settings: Settings | None = None) -> None:
    # 1. 获取项目配置（已包含 default 和 project 优先级的配置）
    if settings is None:
        settings = get_project_settings()
        # ...

    # 2. 解析命令名称
    inproject = inside_project()
    cmds = _get_commands_dict(settings, inproject)
    cmdname = _pop_command_name(argv)
    # ...

    # 3. 获取命令实例
    cmd = cmds[cmdname]
    parser = ScrapyArgumentParser(...)
    
    # 4. ★注入命令内置配置 - 核心代码★
    settings.setdict(cmd.default_settings, priority="command")
    
    # 5. 继续后续流程
    cmd.settings = settings
    cmd.add_options(parser)
    opts, args = parser.parse_known_args(args=argv[1:])
    _run_print_help(parser, cmd.process_options, args, opts)
    # ...
```

**关键代码位置**: `scrapy/cmdline.py:169-215`

### 2.4 注入时机分析

命令内置配置的注入时机非常关键：

```
执行时序（从先到后）：

1. Settings 实例化
   └── 加载 default_settings (priority=0)

2. get_project_settings()
   ├── 从 settings.py 加载项目配置 (priority=20)
   └── 从 SCRAPY_* 环境变量加载 (priority=20)

3. ★ cmd.execute() 中注入命令内置配置 ★
   └── settings.setdict(cmd.default_settings, priority="command")  (priority=10)
   
   ⚠️ 注意：priority=10 < priority=20，所以命令内置配置
      不会覆盖已存在的项目配置！

4. process_options() 处理命令行参数
   └── 从 -s 参数解析 (priority=40，最高优先级)
```

**重要设计意图**：
- 命令内置配置的优先级（10）**低于**项目配置优先级（20）
- 这意味着 `settings.py` 中的配置会覆盖命令的默认配置
- 命令内置配置更像是"命令专属的默认值"，而不是强制覆盖

**示例场景**：
```python
# 如果 settings.py 中配置了：
LOG_ENABLED = True

# 即使 check 命令的 default_settings 声明了：
default_settings = {"LOG_ENABLED": False}

# 最终结果仍是 LOG_ENABLED = True
# 因为 project 优先级 (20) > command 优先级 (10)
```

## 3. 插件配置层（addon，优先级15）

### 3.1 插件配置的两阶段注入

Scrapy 插件系统提供了**两个阶段**的配置注入时机，分别用于不同的场景：

| 阶段 | 方法名 | 调用者 | 时机 | 适用场景 |
|------|--------|--------|------|----------|
| 预爬虫阶段 | `update_pre_crawler_settings` | `AddonManager.load_pre_crawler_settings()` | `CrawlerRunnerBase.__init__()` 中 | 需要提前配置的设置（如 `SPIDER_MODULES`） |
| 爬虫阶段 | `update_settings` | `AddonManager.load_settings()` | `Crawler._apply_settings()` 中 | 普通插件配置 |

### 3.2 预爬虫阶段配置注入

**触发入口**：`CrawlerRunnerBase.__init__()` 中

```python
class CrawlerRunnerBase(ABC):
    def __init__(self, settings: dict[str, Any] | Settings | None = None):
        if isinstance(settings, dict) or settings is None:
            settings = Settings(settings)
        
        # ★预爬虫阶段插件配置注入★
        AddonManager.load_pre_crawler_settings(settings)
        
        self.settings: Settings = settings
        self.spider_loader: SpiderLoaderProtocol = get_spider_loader(settings)
        # ...
```

**关键代码位置**: `scrapy/crawler.py:343-351`

**实现细节**：

```python
class AddonManager:
    @classmethod
    def load_pre_crawler_settings(cls, settings: BaseSettings) -> None:
        """Update early settings that do not require a crawler instance, such as SPIDER_MODULES.

        Similar to the load_settings method, this loads each add-on configured in the
        ``ADDONS`` setting and calls their 'update_pre_crawler_settings' class method if present.
        This method doesn't have access to the crawler instance or the addons list.
        """
        for clspath in build_component_list(settings["ADDONS"]):
            addoncls = load_object(clspath)
            if hasattr(addoncls, "update_pre_crawler_settings"):
                addoncls.update_pre_crawler_settings(settings)
```

**关键代码位置**: `scrapy/addons.py:57-72`

**特点**：
- 这是一个**类方法**，不需要实例化插件
- 调用时机在 `get_spider_loader()` 之前
- 适用于需要影响蜘蛛加载过程的配置（如 `SPIDER_MODULES`）

### 3.3 爬虫阶段配置注入

**触发入口**：`Crawler._apply_settings()` 中

```python
def _apply_settings(self) -> None:
    if self.settings.frozen:
        return

    # ★爬虫阶段插件配置注入★
    self.addons.load_settings(self.settings)
    
    # 组件初始化
    self.stats = load_object(self.settings["STATS_CLASS"])(self)
    # ...
    
    self.extensions = ExtensionManager.from_crawler(self)
    
    # 配置锁定
    self.settings.freeze()
    # ...
```

**关键代码位置**: `scrapy/crawler.py:93-146`

**实现细节**：

```python
class AddonManager:
    def load_settings(self, settings: Settings) -> None:
        """Load add-ons and configurations from a settings object and apply them.

        This will load the add-on for every add-on path in the
        ``ADDONS`` setting and execute their ``update_settings`` methods.
        """
        for clspath in build_component_list(settings["ADDONS"]):
            try:
                addoncls = load_object(clspath)
                addon = build_from_crawler(addoncls, self.crawler)
                
                # ★调用插件的 update_settings 方法★
                if hasattr(addon, "update_settings"):
                    addon.update_settings(settings)
                
                self.addons.append(addon)
            except NotConfigured as e:
                # ...
```

**关键代码位置**: `scrapy/addons.py:25-55`

### 3.4 插件配置的优先级使用

**重要说明**：框架本身**不会自动**为插件配置设置 `priority="addon"`。插件开发者需要在自己的方法中显式指定：

```python
# 插件开发者应该这样写：
class MyAddon:
    @classmethod
    def update_pre_crawler_settings(cls, settings):
        # 显式使用 priority="addon"
        settings.set("MY_SETTING", "value", priority="addon")
    
    def update_settings(self, settings):
        # 显式使用 priority="addon"
        settings.set("ANOTHER_SETTING", "value", priority="addon")
```

**如果不指定优先级**，会使用 `set` 方法的默认值 `"project"`（优先级20），这会导致插件配置优先级高于预期。

### 3.5 插件配置注入时序

```
插件配置注入完整时序：

1. 预爬虫阶段（CrawlerRunnerBase 初始化时）
   ┌─────────────────────────────────────────────────────┐
   │ AddonManager.load_pre_crawler_settings(settings)    │
   │  - 遍历 ADDONS 配置中的插件类                         │
   │  - 调用类方法 update_pre_crawler_settings()         │
   │  - 此时还没有 Crawler 实例                           │
   │  - 可用于配置 SPIDER_MODULES 等早期设置              │
   └─────────────────────────────────────────────────────┘
                ↓
2. 创建 SpiderLoader（依赖预爬虫阶段的配置）
   └── get_spider_loader(settings)
                ↓
3. 创建 Crawler 实例
   ├── settings.copy()
   └── spidercls.update_settings()  # 爬虫自定义配置 (priority=30)
                ↓
4. 爬虫阶段（Crawler._apply_settings 中）
   ┌─────────────────────────────────────────────────────┐
   │ AddonManager.load_settings(settings)                 │
   │  - 实例化插件（需要 crawler 实例）                    │
   │  - 调用实例方法 update_settings()                    │
   │  - 此时可以访问 crawler 的其他组件                    │
   └─────────────────────────────────────────────────────┘
                ↓
5. 初始化其他组件
   ├── ExtensionManager
   ├── Stats
   └── ...
                ↓
6. 配置锁定
   └── settings.freeze()
```

### 3.6 插件配置与其他配置源的优先级关系

```
优先级从低到高：

  0: default      ── 内置默认值
      ↓
 10: command      ── 命令内置配置（如 bench、shell 等）
      ↓
 15: addon        ── 插件配置（两阶段注入）
      ↓
 20: project      ── 项目配置 (settings.py + 环境变量)
      ↓
 30: spider       ── 爬虫 custom_settings
      ↓
 40: cmdline      ── 命令行 -s 参数（最高优先级）
```

## 4. 爬虫级别配置覆盖机制

### 4.1 爬虫自定义配置定义

每个 `Spider` 类都可以定义 `custom_settings` 类属性来指定该爬虫专用的配置：

```python
class Spider(object_ref):
    custom_settings: dict[_SettingsKey, Any] | None = None
```

**关键代码位置**: `scrapy/spiders/__init__.py:42`

### 4.2 配置更新方法

`Spider.update_settings` 是配置合并的核心方法，以 `"spider"` 优先级（值为 30）将自定义配置合并到全局配置：

```python
@classmethod
def update_settings(cls, settings: BaseSettings) -> None:
    settings.setdict(cls.custom_settings or {}, priority="spider")
```

**关键代码位置**: `scrapy/spiders/__init__.py:169-171`

### 4.3 配置合并流程

配置合并发生在 `Crawler` 初始化阶段，遵循以下顺序：

```python
class Crawler:
    def __init__(
        self,
        spidercls: type[Spider],
        settings: dict[str, Any] | Settings | None = None,
        init_reactor: bool = False,
    ):
        if isinstance(settings, dict) or settings is None:
            settings = Settings(settings)

        self.spidercls: type[Spider] = spidercls
        self.settings: Settings = settings.copy()  # 复制传入的配置
        self.spidercls.update_settings(self.settings)  # 合并爬虫自定义配置
        # ... 后续初始化
```

**关键代码位置**: `scrapy/crawler.py:56-71`

### 4.4 嵌套配置的特殊处理

对于字典类型的配置项（如 `DOWNLOADER_MIDDLEWARES`），Scrapy 有特殊处理：

```python
# Settings.__init__ 中的处理
self.setmodule(default_settings, "default")
# 将默认字典提升为 BaseSettings 实例以支持键级优先级
for name, val in self.items():
    if isinstance(val, dict):
        self.set(name, BaseSettings(val, "default"), "default")
```

**关键代码位置**: `scrapy/settings/__init__.py:716-727`

这使得像 `DOWNLOADER_MIDDLEWARES` 这样的组件配置可以基于 `_BASE` 版本进行增量更新：

```python
def getwithbase(self, name: _SettingsKey) -> BaseSettings:
    """获取字典配置及其 _BASE 版本的组合"""
    if not isinstance(name, str):
        raise ValueError(f"Base setting key must be a string, got {name}")
    compbs = BaseSettings()
    compbs.update(self[name + "_BASE"])
    compbs.update(self[name])
    return compbs
```

**关键代码位置**: `scrapy/settings/__init__.py:325-342`

## 5. 配置锁定机制

### 5.1 锁定状态存储

配置对象通过 `frozen` 属性跟踪锁定状态：

```python
class BaseSettings(MutableMapping[_SettingsKey, Any]):
    def __init__(self, values: _SettingsInput = None, priority: int | str = "project"):
        self.frozen: bool = False
        self.attributes: dict[_SettingsKey, SettingsAttribute] = {}
        # ...
```

**关键代码位置**: `scrapy/settings/__init__.py:107-111`

### 5.2 可变性检查

所有修改配置的操作前都会调用 `_assert_mutability` 进行检查：

```python
def _assert_mutability(self) -> None:
    if self.frozen:
        raise TypeError("Trying to modify an immutable Settings object")
```

**关键代码位置**: `scrapy/settings/__init__.py:616-618`

以下方法在修改配置前会触发此检查：
- `set` / `__setitem__`
- `setmodule`
- `update` / `setdict`
- `delete` / `__delitem__`
- `add_to_list` / `remove_from_list`
- `pop`

### 5.3 锁定触发时机

配置锁定发生在 `Crawler._apply_settings` 方法中，这是爬虫启动流程的关键节点：

```python
def _apply_settings(self) -> None:
    if self.settings.frozen:
        return  # 已锁定则跳过

    # 1. 加载插件配置（第二阶段）
    self.addons.load_settings(self.settings)
    
    # 2. 初始化核心组件
    self.stats = load_object(self.settings["STATS_CLASS"])(self)
    # ... 其他组件初始化
    
    # 3. 初始化扩展管理器
    self.extensions = ExtensionManager.from_crawler(self)
    
    # 4. ★锁定配置 - 禁止后续修改★
    self.settings.freeze()
    
    # 5. 输出被覆盖的配置信息
    d = dict(overridden_settings(self.settings))
    logger.info(
        "Overridden settings:\n%(settings)s", {"settings": pprint.pformat(d)}
    )
```

**关键代码位置**: `scrapy/crawler.py:93-151`

### 5.4 锁定方法实现

`freeze` 方法和 `frozencopy` 方法提供了锁定功能：

```python
def freeze(self) -> None:
    """禁用对当前配置的进一步修改"""
    self.frozen = True

def frozencopy(self) -> Self:
    """返回当前配置的不可变副本"""
    copy = self.copy()
    copy.freeze()
    return copy
```

**关键代码位置**: `scrapy/settings/__init__.py:632-650`

## 6. 配置系统架构总结

### 6.1 核心类层次结构

```
BaseSettings (MutableMapping)
    ├── 基础功能：优先级存储、锁定机制、类型转换
    └── Settings (子类)
            └── 初始化时自动加载 default_settings

ScrapyCommand
    └── default_settings 类属性声明命令专属配置

AddonManager
    ├── load_pre_crawler_settings() - 预爬虫阶段配置注入
    └── load_settings() - 爬虫阶段配置注入

Spider
    └── update_settings() - 爬虫自定义配置合并
```

### 6.2 完整配置加载时序图

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                      完整配置加载时序（从低到高优先级）                         │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  优先级 0: default                                                           │
│  ┌──────────────────────────────────────────────────────────────────────┐ │
│  │ Settings.__init__()                                                    │ │
│  │   └── self.setmodule(default_settings, "default")                     │ │
│  │       └── 加载 scrapy.settings.default_settings 模块                   │ │
│  └──────────────────────────────────────────────────────────────────────┘ │
│                                      ↓                                       │
│  优先级 20: project                                                          │
│  ┌──────────────────────────────────────────────────────────────────────┐ │
│  │ get_project_settings()                                                 │ │
│  │   ├── settings.setmodule(settings_module_path, "project")            │ │
│  │   │       └── 从 settings.py 加载项目配置                              │ │
│  │   └── settings.setdict(scrapy_envvars, "project")                     │ │
│  │           └── 从 SCRAPY_* 环境变量加载                                 │ │
│  └──────────────────────────────────────────────────────────────────────┘ │
│                                      ↓                                       │
│  优先级 10: command  ⚠️ 注意：优先级低于 project                           │
│  ┌──────────────────────────────────────────────────────────────────────┐ │
│  │ cmdline.execute()                                                      │ │
│  │   └── settings.setdict(cmd.default_settings, priority="command")      │ │
│  │       示例：                                                            │ │
│  │       - bench: LOG_LEVEL, LOGSTATS_INTERVAL, CLOSESPIDER_TIMEOUT     │ │
│  │       - shell: DUPEFILTER_CLASS, KEEP_ALIVE, LOGSTATS_INTERVAL       │ │
│  │       - check/edit/...: LOG_ENABLED = False                            │ │
│  │                                                                         │ │
│  │       ⚠️  由于 priority=10 < 20，这些配置不会覆盖 settings.py 中的设置 │ │
│  └──────────────────────────────────────────────────────────────────────┘ │
│                                      ↓                                       │
│  优先级 40: cmdline                                                          │
│  ┌──────────────────────────────────────────────────────────────────────┐ │
│  │ ScrapyCommand.process_options()                                        │ │
│  │   └── self.settings.setdict(arglist_to_dict(opts.set), "cmdline")     │ │
│  │       └── 解析 -s NAME=VALUE 命令行参数                                │ │
│  └──────────────────────────────────────────────────────────────────────┘ │
│                                      ↓                                       │
│  优先级 15: addon (第一阶段 - 预爬虫)                                        │
│  ┌──────────────────────────────────────────────────────────────────────┐ │
│  │ CrawlerRunnerBase.__init__()                                           │ │
│  │   └── AddonManager.load_pre_crawler_settings(settings)                 │ │
│  │       └── 调用插件类方法 update_pre_crawler_settings()                 │ │
│  │           用于配置 SPIDER_MODULES 等早期设置                           │ │
│  └──────────────────────────────────────────────────────────────────────┘ │
│                                      ↓                                       │
│  优先级 30: spider                                                           │
│  ┌──────────────────────────────────────────────────────────────────────┐ │
│  │ Crawler.__init__()                                                      │ │
│  │   ├── self.settings = settings.copy()                                   │ │
│  │   └── self.spidercls.update_settings(self.settings)                    │ │
│  │           └── settings.setdict(custom_settings, "spider")             │ │
│  └──────────────────────────────────────────────────────────────────────┘ │
│                                      ↓                                       │
│  优先级 15: addon (第二阶段 - 爬虫)                                          │
│  ┌──────────────────────────────────────────────────────────────────────┐ │
│  │ Crawler._apply_settings()                                              │ │
│  │   └── self.addons.load_settings(self.settings)                         │ │
│  │       └── 调用插件实例方法 update_settings()                            │ │
│  │           ⚠️ 插件需要自己指定 priority="addon"                         │ │
│  └──────────────────────────────────────────────────────────────────────┘ │
│                                      ↓                                       │
│  配置锁定点                                                                   │
│  ┌──────────────────────────────────────────────────────────────────────┐ │
│  │ self.settings.freeze()                                                  │ │
│  │   └── 所有后续修改尝试都会抛出 TypeError                                │ │
│  └──────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 6.3 优先级与注入时机对应关系

| 优先级 | 名称 | 数值 | 注入时机 | 关键方法 |
|--------|------|------|----------|----------|
| 最低 | default | 0 | `Settings.__init__` | `setmodule(default_settings, "default")` |
| ↓ | command | 10 | `cmdline.execute()` | `settings.setdict(cmd.default_settings, "command")` |
| ↓ | addon | 15 | 两阶段 | `load_pre_crawler_settings` / `load_settings` |
| ↓ | project | 20 | `get_project_settings()` | `setmodule(settings_module, "project")` |
| ↓ | spider | 30 | `Crawler.__init__` | `spider.update_settings(settings)` |
| 最高 | cmdline | 40 | `process_options()` | `settings.setdict(..., "cmdline")` |

> **注意**：`command` 优先级（10）低于 `project` 优先级（20），但注入时机在 `project` 之后。这是因为优先级数值才是决定覆盖关系的关键，而非注入顺序。

### 6.4 关键配置方法速查

| 方法 | 功能 | 优先级参数 |
|------|------|------------|
| `set(name, value, priority)` | 设置单个配置项 | 支持 |
| `setdict(values, priority)` | 批量设置配置 | 支持 |
| `setmodule(module, priority)` | 从模块加载配置 | 支持 |
| `update(values, priority)` | 通用更新方法 | 支持 |
| `get(name, default)` | 获取配置值 | - |
| `getpriority(name)` | 获取配置优先级 | - |
| `freeze()` | 锁定配置 | - |
| `frozencopy()` | 创建锁定副本 | - |

## 7. 代码引用速查

| 功能 | 文件位置 | 行号 |
|------|----------|------|
| 优先级定义 | `scrapy/settings/__init__.py` | 35-42 |
| SettingsAttribute 类 | `scrapy/settings/__init__.py` | 56-80 |
| BaseSettings 核心类 | `scrapy/settings/__init__.py` | 83-703 |
| Settings 类 | `scrapy/settings/__init__.py` | 705-728 |
| 配置锁定检查 | `scrapy/settings/__init__.py` | 616-618 |
| 锁定方法 | `scrapy/settings/__init__.py` | 632-650 |
| ScrapyCommand 基类 | `scrapy/commands/__init__.py` | 27-33 |
| 命令配置注入 | `scrapy/cmdline.py` | 200 |
| AddonManager 类 | `scrapy/addons.py` | 18-72 |
| 预爬虫插件配置 | `scrapy/crawler.py` | 347 |
| 爬虫阶段插件配置 | `scrapy/crawler.py` | 97 |
| 蜘蛛自定义配置 | `scrapy/spiders/__init__.py` | 42, 169-171 |
| Crawler 配置合并 | `scrapy/crawler.py` | 56-71 |
| Crawler 配置锁定 | `scrapy/crawler.py` | 93-146 |
| 项目配置加载 | `scrapy/utils/project.py` | 66-91 |
| 默认配置 | `scrapy/settings/default_settings.py` | 全文 |
