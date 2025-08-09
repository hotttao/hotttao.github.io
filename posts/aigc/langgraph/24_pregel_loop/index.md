# pregel loop


## 1. PregelLoop
```bash
# 提问
上面是 Langgraph PregelLoop 的定义，请给我解释一下 PregelLoop 的语义，并以表格的形式列举 PregelLoop 的属性，并以另一个表格列举 PregelLoop 的方法
```
### 1.1 PregelLoop 语义

`PregelLoop` 是 LangGraph 中实现 Pregel 模式的核心类之一，承担了图执行调度器的角色。它负责以超步（superstep）方式逐步调度图中的节点（`PregelNode`），并在每个 tick 中处理任务的生成、执行、中断判断、检查点记录、流写入以及缓存等一系列关键操作。PregelLoop 应该是 LangGraph 中最复杂的类，也是最核心的类。



| 角色      | 说明                                                         |
| ------- | ---------------------------------------------------------- |
| 图调度器    | 管理 Pregel 模式下的图执行生命周期（构建任务 → 执行任务 → 收集结果 → 写入 checkpoint）。 |
| 状态管理者   | 保存当前步骤、节点状态、通道值、缓存等，用于恢复或继续执行。                             |
| 检查点协作者  | 协调持久化逻辑（通过 `checkpointer`）将中间状态保存至外部系统（如 DB、文件系统）。         |
| 中断机制协调者 | 提供 `interrupt_before` 与 `interrupt_after` 中断控制机制。          |
| 输出协调者   | 控制哪些任务的输出将通过 `stream` 发出，以及是否触发结果值输出。                      |


### 1.3 PregelLoop 属性

| 属性名                                         | 类型                                                  | 说明                                     |
| ------------------------------------------- | --------------------------------------------------- | -------------------------------------- |
| `config`                                    | `RunnableConfig`                                    | 当前执行配置（包含命名空间、恢复标记等元数据）。               |
| `store`                                     | `BaseStore \| None`                                 | 存储层（用于读取通道值等）。                         |
| `stream`                                    | `StreamProtocol \| None`                            | 流系统（用于发送中间状态，如 task、values）。           |
| `step`, `stop`                              | `int`                                               | 当前执行步数、最多步数。                           |
| `input`                                     | `Any \| None`                                       | 初始输入值。                                 |
| `cache`                                     | `BaseCache[WritesT] \| None`                        | 缓存系统。                                  |
| `checkpointer`                              | `BaseCheckpointSaver \| None`                       | 检查点保存器。                                |
| `nodes`                                     | `Mapping[str, PregelNode]`                          | 图中所有节点定义。                              |
| `specs`                                     | `Mapping[str, BaseChannel \| ManagedValueSpec]`     | 通道定义集合。                                |
| `input_keys`, `output_keys`, `stream_keys`  | `str \| list[str]`                                  | 指定输入、输出、流通道 key。                       |
| `skip_done_tasks`                           | `bool`                                              | 是否跳过已完成任务（用于恢复场景）。                     |
| `is_nested`                                 | `bool`                                              | 是否为子图执行（由 `config` 中是否含 task_id 判定）。  |
| `manager`                                   | `AsyncParentRunManager \| ParentRunManager \| None` | 执行管理器。                                 |
| `interrupt_before`, `interrupt_after`       | `All \| Sequence[str]`                              | 中断点配置。                                 |
| `durability`                                | `Durability`                                        | 是否启用持久化机制。                             |
| `retry_policy`                              | `Sequence[RetryPolicy]`                             | 重试策略。                                  |
| `cache_policy`                              | `CachePolicy \| None`                               | 缓存策略。                                  |
| `checkpointer_get_next_version`             | `GetNextVersion`                                    | 获取通道版本号的函数。                            |
| `checkpointer_put_writes`                   | `Callable \| None`                                  | 写入 checkpoint 的方法。                     |
| `checkpointer_put_writes_accepts_task_path` | `bool`                                              | 是否支持传入 `task.path` 参数。                 |
| `_checkpointer_put_after_previous`          | `Callable \| None`                                  | checkpoint 之后的异步提交钩子。                  |
| `_migrate_checkpoint`                       | `Callable \| None`                                  | 检查点迁移逻辑（通常用于版本迁移）。                     |
| `submit`                                    | `Submit`                                            | 异步提交方法（通常是线程池或事件循环封装）。                 |
| `channels`                                  | `Mapping[str, BaseChannel]`                         | 所有运行中的通道状态。                            |
| `managed`                                   | `ManagedValueMapping`                               | 管理的通道值（含版本）。                           |
| `checkpoint`                                | `Checkpoint`                                        | 当前执行中的检查点状态。                           |
| `checkpoint_id_saved`                       | `str`                                               | 最近保存的 checkpoint ID。                   |
| `checkpoint_ns`                             | `tuple[str, ...]`                                   | 当前命名空间（嵌套图结构用）。                        |
| `checkpoint_config`                         | `RunnableConfig`                                    | 用于 checkpoint 的配置快照。                   |
| `checkpoint_metadata`                       | `CheckpointMetadata`                                | checkpoint 的元信息。                       |
| `checkpoint_pending_writes`                 | `list[PendingWrite]`                                | 待保存的写入项。                               |
| `checkpoint_previous_versions`              | `dict[str, str \| float \| int]`                    | 上一个 checkpoint 的通道版本信息。                |
| `prev_checkpoint_config`                    | `RunnableConfig \| None`                            | 上一次的 config 快照（用于调试对比）。                |
| `status`                                    | `Literal[...]`                                      | 当前状态标识：input、pending、done、interrupt、等。 |
| `tasks`                                     | `dict[str, PregelExecutableTask]`                   | 当前调度周期生成的所有任务。                         |
| `output`                                    | `dict[str, Any] \| None`                            | 最终输出。                                  |
| `updated_channels`                          | `set[str] \| None`                                  | 最新被写入的通道集合。                            |
| `trigger_to_nodes`                          | `Mapping[str, Sequence[str]]`                       | 通道变更 -> 触发节点映射关系。                      |


下面是 PregelLoop 初始化的代码，代码比较长:

```python
# Pregel 初始化 SyncPregelLoop 的代码，可以帮助我们理解 SyncPregelLoop 与 Pregel 的关系
with SyncPregelLoop(
    input,
    stream=StreamProtocol(stream.put, stream_modes),
    config=config,
    store=store,
    cache=cache,
    checkpointer=checkpointer,
    nodes=self.nodes,
    specs=self.channels, 
    output_keys=output_keys,
    input_keys=self.input_channels,
    stream_keys=self.stream_channels_asis,
    interrupt_before=interrupt_before_,
    interrupt_after=interrupt_after_,
    manager=run_manager,
    durability=durability_,
    trigger_to_nodes=self.trigger_to_nodes,
    migrate_checkpoint=self._migrate_checkpoint,
    retry_policy=self.retry_policy,
    cache_policy=self.cache_policy,
) as loop:
  pass

class PregelLoop:
    config: RunnableConfig
    store: BaseStore | None
    stream: StreamProtocol | None
    step: int
    stop: int

    input: Any | None
    cache: BaseCache[WritesT] | None
    checkpointer: BaseCheckpointSaver | None
    nodes: Mapping[str, PregelNode]
    specs: Mapping[str, BaseChannel | ManagedValueSpec]  # 所有 channels
    input_keys: str | Sequence[str]
    output_keys: str | Sequence[str]
    stream_keys: str | Sequence[str]
    skip_done_tasks: bool
    is_nested: bool
    manager: None | AsyncParentRunManager | ParentRunManager
    interrupt_after: All | Sequence[str]
    interrupt_before: All | Sequence[str]
    durability: Durability
    retry_policy: Sequence[RetryPolicy]
    cache_policy: CachePolicy | None

    checkpointer_get_next_version: GetNextVersion
    checkpointer_put_writes: Callable[[RunnableConfig, WritesT, str], Any] | None
    checkpointer_put_writes_accepts_task_path: bool
    _checkpointer_put_after_previous: (
        Callable[
            [
                concurrent.futures.Future | None,
                RunnableConfig,
                Checkpoint,
                str,
                ChannelVersions,
            ],
            Any,
        ]
        | None
    )
    _migrate_checkpoint: Callable[[Checkpoint], None] | None
    submit: Submit
    channels: Mapping[str, BaseChannel]
    managed: ManagedValueMapping
    checkpoint: Checkpoint
    checkpoint_id_saved: str
    checkpoint_ns: tuple[str, ...]
    checkpoint_config: RunnableConfig
    checkpoint_metadata: CheckpointMetadata
    checkpoint_pending_writes: list[PendingWrite]
    checkpoint_previous_versions: dict[str, str | float | int]
    prev_checkpoint_config: RunnableConfig | None

    status: Literal[
        "input",
        "pending",
        "done",
        "interrupt_before",
        "interrupt_after",
        "out_of_steps",
    ]
    tasks: dict[str, PregelExecutableTask]
    output: None | dict[str, Any] | Any = None
    updated_channels: set[str] | None = None

    # public

    def __init__(
        self,
        input: Any | None,
        *,
        stream: StreamProtocol | None,
        config: RunnableConfig,
        store: BaseStore | None,
        cache: BaseCache | None,
        checkpointer: BaseCheckpointSaver | None,
        nodes: Mapping[str, PregelNode],
        specs: Mapping[str, BaseChannel | ManagedValueSpec],
        input_keys: str | Sequence[str],
        output_keys: str | Sequence[str],
        stream_keys: str | Sequence[str],
        trigger_to_nodes: Mapping[str, Sequence[str]],
        durability: Durability,
        interrupt_after: All | Sequence[str] = EMPTY_SEQ,
        interrupt_before: All | Sequence[str] = EMPTY_SEQ,
        manager: None | AsyncParentRunManager | ParentRunManager = None,
        migrate_checkpoint: Callable[[Checkpoint], None] | None = None,
        retry_policy: Sequence[RetryPolicy] = (),
        cache_policy: CachePolicy | None = None,
    ) -> None:
        # 处理传入的参数 
        self.stream = stream
        self.config = config
        self.store = store
        self.step = 0
        self.stop = 0
        self.input = input
        self.checkpointer = checkpointer
        self.cache = cache
        self.nodes = nodes
        self.specs = specs
        self.input_keys = input_keys
        self.output_keys = output_keys
        self.stream_keys = stream_keys
        self.interrupt_after = interrupt_after
        self.interrupt_before = interrupt_before
        self.manager = manager
        # 从 config 中提取的参数
        self.is_nested = CONFIG_KEY_TASK_ID in self.config.get(CONF, {})
        self.skip_done_tasks = CONFIG_KEY_CHECKPOINT_ID not in config[CONF]
        self._migrate_checkpoint = migrate_checkpoint
        self.trigger_to_nodes = trigger_to_nodes
        self.retry_policy = retry_policy
        self.cache_policy = cache_policy
        self.durability = durability
        if self.stream is not None and CONFIG_KEY_STREAM in config[CONF]:
            self.stream = DuplexStream(self.stream, config[CONF][CONFIG_KEY_STREAM])
        scratchpad: PregelScratchpad | None = config[CONF].get(CONFIG_KEY_SCRATCHPAD)
        if isinstance(scratchpad, PregelScratchpad):
            # if count is > 0, append to checkpoint_ns
            # if count is 0, leave as is
            if cnt := scratchpad.subgraph_counter():
                self.config = patch_configurable(
                    self.config,
                    {
                        CONFIG_KEY_CHECKPOINT_NS: NS_SEP.join(
                            (
                                config[CONF][CONFIG_KEY_CHECKPOINT_NS],
                                str(cnt),
                            )
                        )
                    },
                )
        # 如果不存在 task_id，说明是 root graph，重置 checkpoint_ns 和 checkpoint_id

        if not self.is_nested and config[CONF].get(CONFIG_KEY_CHECKPOINT_NS):
            self.config = patch_configurable(
                self.config,
                {CONFIG_KEY_CHECKPOINT_NS: "", CONFIG_KEY_CHECKPOINT_ID: None},
            )
        if (
            CONFIG_KEY_CHECKPOINT_MAP in self.config[CONF]
            and self.config[CONF].get(CONFIG_KEY_CHECKPOINT_NS)
            in self.config[CONF][CONFIG_KEY_CHECKPOINT_MAP]
        ):
            self.checkpoint_config = patch_configurable(
                self.config,
                {
                    CONFIG_KEY_CHECKPOINT_ID: self.config[CONF][
                        CONFIG_KEY_CHECKPOINT_MAP
                    ][self.config[CONF][CONFIG_KEY_CHECKPOINT_NS]]
                },
            )
        else:
            self.checkpoint_config = self.config
        if thread_id := self.checkpoint_config[CONF].get(CONFIG_KEY_THREAD_ID):
            if not isinstance(thread_id, str):
                self.checkpoint_config = patch_configurable(
                    self.checkpoint_config,
                    {CONFIG_KEY_THREAD_ID: str(thread_id)},
                )
        self.checkpoint_ns = (
            tuple(cast(str, self.config[CONF][CONFIG_KEY_CHECKPOINT_NS]).split(NS_SEP))
            if self.config[CONF].get(CONFIG_KEY_CHECKPOINT_NS)
            else ()
        )
        self.prev_checkpoint_config = None
```

很大一部分是从 RunnableConfig 中提取相关配置。这里对相关的配置项做一个整理:

```python
{
    "__pregel_task_id": "task_id",    # is_nested = CONFIG_KEY_TASK_ID in configurable，判断当前是不是子图
    "checkpoint_id": "checkpoint_id", # skip_done_tasks = CONFIG_KEY_CHECKPOINT_ID not in configurable
    "checkpoint_ns": "checkpoint_ns",
    "thread_id": "thread_id",
    "__pregel_stream": "StreamProtocol", # 合并 stream
    "__pregel_scratchpad": "scratchpad", # 重新配置 __pregel_checkpoint_ns/subgraph_counter
}
```

#### StreamProtocol
StreamProtocol 是一个带有“输出模式”的流处理协议接口，它包装了一个可调用函数，用于处理流数据（如日志、模型输出、状态更新等）。

```python
StreamChunk = tuple[tuple[str, ...], str, Any]

class StreamProtocol:
    __slots__ = ("modes", "__call__")

    modes: set[StreamMode]

    __call__: Callable[[Self, StreamChunk], None]

    def __init__(
        self,
        __call__: Callable[[StreamChunk], None],
        modes: set[StreamMode],
    ) -> None:
        self.__call__ = cast(Callable[[Self, StreamChunk], None], __call__)
        self.modes = modes

stream=StreamProtocol(stream.put, stream_modes)
stream(stream_chunk)
```


### 1.4 PregelLoop 方法

PregelLoop 类的方法都比较复杂，接下来我们一一讲解。

| 方法名                                            | 作用                       | 是否私有 | 关键点说明                                     |
| ---------------------------------------------- | ------------------------ | ---- | ----------------------------------------- |
| `__init__`                                     | 初始化所有运行环境、检查点、通道等        | ❌    | 初始化复杂，含 scratchpad、resume、namespace 重写等逻辑 |
| `tick`                                         | 执行一次迭代：构建任务、判断中断、执行缓存    | ❌    | Pregel 超步逻辑的关键入口                          |
| `after_tick`                                   | 完成迭代后的尾处理：写入、emit、更新状态   | ❌    | 会在每次 tick 成功后调用                           |
| `_first`                                       | 初始化输入阶段的 apply 逻辑        | ✅    | 在图起点或恢复时触发                                |
| `_put_checkpoint`                              | 执行 checkpoint 写入         | ✅    | 控制 step 增长、metadata 更新                    |
| `_put_pending_writes`                          | 执行所有缓存写入提交               | ✅    | 提交到 checkpointer                          |
| `put_writes`                                   | 写入一个 task 的 writes 数据    | ❌    | 包含 checkpoint 保存逻辑                        |
| `_match_writes`                                | 将写入匹配到已有任务               | ✅    | 用于 skip\_done\_task 时将缓存应用到 task          |
| `accept_push`                                  | 接收 PUSH 操作并构建新的任务        | ❌    | 从某个 task 中激发出一个新的 task（如 subgraph）        |
| `match_cached_writes` / `amatch_cached_writes` | 匹配缓存中的 writes（未实现）       | ❌    | 接口预留                                      |
| `_suppress_interrupt`                          | 处理 GraphInterrupt 时的行为控制 | ✅    | 主要用于嵌套图控制                                 |
| `output_writes`                                | 发出任务写入的 stream 事件        | ❌    | 控制是否输出、是否过滤 hidden                        |
| `_emit`                                        | 执行 stream 写入             | ✅    | 支持 debug 模式映射                             |


## 2. SyncPregelLoop ContextManager 实现
PregelLoop 一部分参数是在上下文管理器中实现的。上下文管理器实现在 PregelLoop 子类上，这里我们以 SyncPregelLoop 为例。

### 2.1 SyncPregelLoop 初始化
初始化参数完成 checkpointer 相关参数的初始化，并设置 `self.stack`

```python
class SyncPregelLoop(PregelLoop, AbstractContextManager):
    def __init__(
        self,
        input: Any | None,
        *,
        stream: StreamProtocol | None,
        config: RunnableConfig,
        store: BaseStore | None,
        cache: BaseCache | None,
        checkpointer: BaseCheckpointSaver | None,
        nodes: Mapping[str, PregelNode],
        specs: Mapping[str, BaseChannel | ManagedValueSpec],
        trigger_to_nodes: Mapping[str, Sequence[str]],
        durability: Durability,
        manager: None | AsyncParentRunManager | ParentRunManager = None,
        interrupt_after: All | Sequence[str] = EMPTY_SEQ,
        interrupt_before: All | Sequence[str] = EMPTY_SEQ,
        input_keys: str | Sequence[str] = EMPTY_SEQ,
        output_keys: str | Sequence[str] = EMPTY_SEQ,
        stream_keys: str | Sequence[str] = EMPTY_SEQ,
        migrate_checkpoint: Callable[[Checkpoint], None] | None = None,
        retry_policy: Sequence[RetryPolicy] = (),
        cache_policy: CachePolicy | None = None,
    ) -> None:
        super().__init__(
            input,
            stream=stream,
            config=config,
            checkpointer=checkpointer,
            cache=cache,
            store=store,
            nodes=nodes,
            specs=specs,
            input_keys=input_keys,
            output_keys=output_keys,
            stream_keys=stream_keys,
            interrupt_after=interrupt_after,
            interrupt_before=interrupt_before,
            manager=manager,
            migrate_checkpoint=migrate_checkpoint,
            trigger_to_nodes=trigger_to_nodes,
            retry_policy=retry_policy,
            cache_policy=cache_policy,
            durability=durability,
        )
        self.stack = ExitStack()
        if checkpointer:
            # 获取 checkpoint 下一个版本号
            self.checkpointer_get_next_version = checkpointer.get_next_version
            # put_writes 函数用于存储与任务关联的中间产出
            self.checkpointer_put_writes = checkpointer.put_writes
            # 检查put_writes是否接收 task_path 参数
            self.checkpointer_put_writes_accepts_task_path = (
                signature(checkpointer.put_writes).parameters.get("task_path")
                is not None
            )
        else:
            self.checkpointer_get_next_version = increment
            self._checkpointer_put_after_previous = None  # type: ignore[assignment]
            self.checkpointer_put_writes = None
            self.checkpointer_put_writes_accepts_task_path = False
```

### 2.2 ExitStack

`ExitStack` 是 contextlib 提供的 **可组合的上下文管理器堆栈**，我们通过一个示例来了解他的用法:


```python
# 传统写法（静态嵌套）
with open("a.txt") as f1:
    with open("b.txt") as f2:
        ...

# 使用 `ExitStack`（动态注册）
from contextlib import ExitStack

with ExitStack() as stack:
    f1 = stack.enter_context(open("a.txt"))
    f2 = stack.enter_context(open("b.txt"))
    ...
# 所有资源都会在这里自动关闭
```

这在“资源数量不确定”时非常有用，例如：

```python
with ExitStack() as stack:
    files = [stack.enter_context(open(name)) for name in file_list]
```

### 2.3 ContextManager
讲解完 SyncPregelLoop 初始化，我们来讲解 SyncPregelLoop 的上下文管理器的实现。

`__enter__` 主要完成以下几个任务:
1. 加载 checkpoint，load 其参数
2. 初始化 self.submit 参数
3. channels_from_checkpoint 从checkpoint 中恢复 channel 的值

```python
    def __enter__(self) -> Self:
        # 1. load checkpoint
        if self.checkpointer:
            saved = self.checkpointer.get_tuple(self.checkpoint_config)
        else:
            saved = None
        if saved is None:
            # 设置默认 checkpoint
            saved = CheckpointTuple(
                self.checkpoint_config, empty_checkpoint(), {"step": -2}, None, []
            )
        # _migrate_checkpoint 是传入的 Pregel._migrate_checkpoint 方法
        # 在框架检测到某个 Checkpoint 是旧格式（version < 4）时，将其字段结构升级为当前的新版本格式，以便继续使用
        elif self._migrate_checkpoint is not None:
            self._migrate_checkpoint(saved.checkpoint)
        
        # 按照 checkpoint 的 config 为最高优先级，重载 config
        self.checkpoint_config = {
            **self.checkpoint_config,
            **saved.config,
            CONF: {
                CONFIG_KEY_CHECKPOINT_NS: "",
                **self.checkpoint_config.get(CONF, {}),
                **saved.config.get(CONF, {}),
            },
        }
        self.prev_checkpoint_config = saved.parent_config
        self.checkpoint_id_saved = saved.checkpoint["id"]
        self.checkpoint = saved.checkpoint
        self.checkpoint_metadata = saved.metadata
        self.checkpoint_pending_writes = (
            # task_id,channel,value
            [(str(tid), k, v) for tid, k, v in saved.pending_writes]
            if saved.pending_writes is not None
            else []
        )
        # 2. BackgroundExecutor 是线程池，self.submit = BackgroundExecutor.submit 方法
        self.submit = self.stack.enter_context(BackgroundExecutor(self.config)) 
        # channels_from_checkpoint 用于从 checkpoint 中恢复 channel 的值。
        self.channels, self.managed = channels_from_checkpoint(
            self.specs, self.checkpoint
        )
        self.stack.push(self._suppress_interrupt)
        self.status = "input"
        self.step = self.checkpoint_metadata["step"] + 1
        self.stop = self.step + self.config["recursion_limit"] + 1
        self.checkpoint_previous_versions = self.checkpoint["channel_versions"].copy()
        self.updated_channels = self._first(input_keys=self.input_keys)

        return self

    def __exit__(
        self,
        exc_type: type[BaseException] | None,
        exc_value: BaseException | None,
        traceback: TracebackType | None,
    ) -> bool | None:
        # unwind stack
        return self.stack.__exit__(exc_type, exc_value, traceback)
```

#### self.submit
```python
self.stack.enter_context(BackgroundExecutor(self.config))
```
这段代码会调用 BackgroundExecutor 的 `__enter__` 方法，`BackgroundExecutor.__enter__` 返回的是其 submit 方法。


#### channels_from_checkpoint
channels_from_checkpoint 用于从 checkpoint 中恢复 channel 的值。

```python
def channels_from_checkpoint(
    specs: Mapping[str, BaseChannel | ManagedValueSpec],
    checkpoint: Checkpoint,
) -> tuple[Mapping[str, BaseChannel], ManagedValueMapping]:
    """Get channels from a checkpoint."""
    channel_specs: dict[str, BaseChannel] = {}
    managed_specs: dict[str, ManagedValueSpec] = {}
    for k, v in specs.items():
        if isinstance(v, BaseChannel):
            channel_specs[k] = v
        else:
            managed_specs[k] = v
    return (
        {
            # load checkpoint 中的 channel 值
            k: v.from_checkpoint(checkpoint["channel_values"].get(k, MISSING))
            for k, v in channel_specs.items()
        },
        managed_specs,
    )
```

至此我们基本上讲解完了 PregelLoop 的初始化，接下来我们来讲解 PregelLoop 的方法。前面我们讲解 Pregel 代码时已经分析了 stream 方法的基本流程。我们将按照下面的顺序介绍 PregelLoop 中的方法:
1. tick
2. match_cached_writes
3. output_writes
4. after_tick
5. _put_checkpoint_fut

## 3. tick
```python
    def tick(self) -> bool:
        """Execute a single iteration of the Pregel loop.

        Args:
            input_keys: The key(s) to read input from.

        Returns:
            True if more iterations are needed.
        """

        # check if iteration limit is reached
        if self.step > self.stop:
            self.status = "out_of_steps"
            return False

        # prepare next tasks
        self.tasks = prepare_next_tasks(
            self.checkpoint, 
            self.checkpoint_pending_writes,
            self.nodes,
            self.channels,
            self.managed,
            self.config,
            self.step,
            self.stop,
            for_execution=True,
            manager=self.manager,
            store=self.store,
            checkpointer=self.checkpointer,
            trigger_to_nodes=self.trigger_to_nodes,
            updated_channels=self.updated_channels,
            retry_policy=self.retry_policy,
            cache_policy=self.cache_policy,
        )

        # produce debug output
        if self._checkpointer_put_after_previous is not None:
            self._emit(
                "checkpoints",
                map_debug_checkpoint,
                {
                    **self.checkpoint_config,
                    CONF: {
                        **self.checkpoint_config[CONF],
                        CONFIG_KEY_CHECKPOINT_ID: self.checkpoint["id"],
                    },
                },
                self.channels,
                self.stream_keys,
                self.checkpoint_metadata,
                self.tasks.values(),
                self.checkpoint_pending_writes,
                self.prev_checkpoint_config,
                self.output_keys,
            )

        # if no more tasks, we're done
        if not self.tasks:
            self.status = "done"
            return False

        # if there are pending writes from a previous loop, apply them
        if self.skip_done_tasks and self.checkpoint_pending_writes:
            self._match_writes(self.tasks)

        # before execution, check if we should interrupt
        if self.interrupt_before and should_interrupt(
            self.checkpoint, self.interrupt_before, self.tasks.values()
        ):
            self.status = "interrupt_before"
            raise GraphInterrupt()

        # produce debug output
        self._emit("tasks", map_debug_tasks, self.tasks.values())

        # print output for any tasks we applied previous writes to
        for task in self.tasks.values():
            if task.writes:
                self.output_writes(task.id, task.writes, cached=True)

        return True
```

tick 方法中，会调用 prepare_next_tasks 方法生成当前 step 的 task，prepare_next_tasks 中的入参来源如下:
1. self.checkpoint:
    - `Loop.__enter__` 方法中，会使用 configurable 中保存的 CONFIG_KEY_CHECKPOINT_ID 重新load checkpoint。
    - 如果没有 checkpoint，使用 empty_checkpoint() 生成一个空的 checkpoint
2. self.checkpoint_pending_writes:
    - 从恢复的 checkpoint 读取，默认为空
3. self.nodes: Loop 初始化传入，`Mapping[str, PregelNode]`
4. self.channels: 
    - Loop 初始化传入 `spec=Mapping[str, BaseChannel | ManagedValueSpec]`
    - `Loop.__enter__` 方法中，会调用 channels_from_checkpoint 方法从 checkpoint 中恢复 channel 的值，并提取出 ManagedValueSpec
5. self.managed 同 self.channels
6. self.config: Loop 初始化传入，并在调用过程中更新
7. self.step: Loop 初始化为 0，`Loop.__enter__` 中重置为 `self.checkpoint_metadata["step"] + 1`
8. self.stop: Loop 初始化为 0，`Loop.__enter__` 中重置为`self.step + self.config["recursion_limit"] + 1`
9. self.manager: Loop 初始化传入
10. self.store: Loop 初始化传入
10. self.checkpointer: Loop 初始化传入
11. self.trigger_to_nodes: Loop 初始化传入 `Mapping[str, Sequence[str]]` channel 更新要触发的节点映射
13. self.updated_channels
    - `Loop.__enter__` 方法中，会调用 `self._first(input_keys=self.input_keys)` 对 updated_channels 做初始化
    - `after_tick` 方法中，会调用 `self.updated_channels = apply_writes)` 完成 channel 的更新，并返回更新过的 channel
10. self.retry_policy: Loop 初始化传入
10. self.cache_policy: Loop 初始化传入

现在我们来看与 updated_channels 相关的 `loop._first` 方法，

## 4. _first 方法
_first 是 Pregel 图首次调度（或恢复）的核心入口。 


### 4.1 _first 方法签名说明


```python
def _first(self, *, input_keys: str | Sequence[str]) -> set[str] | None:
```

* `input_keys`: 当前节点接收的输入通道名（channel key），用于解析输入。
* 返回值是 `set[str]` 或 `None`，表示有哪些通道被更新（用于后续调度）。

---

### 4.2 _first 执行逻辑

按照执行逻辑，函数大致分为 6 步：

#### **判断是否处于恢复状态（is\_resuming）**
➡️ **作用**：决定是否要跳过输入写入，直接恢复。

```python
        configurable = self.config.get(CONF, {})
        is_resuming = bool(self.checkpoint["channel_versions"]) and bool(
            configurable.get(
                CONFIG_KEY_RESUMING,
                self.input is None
                or isinstance(self.input, Command)
                or (
                    # self.is_nested = CONFIG_KEY_TASK_ID in self.config.get(CONF, {})
                    # not 表示非子图执行
                    not self.is_nested
                    # 配置的 run_id 和从恢复的 checkpoint 中的 run_id 一致
                    and self.config.get("metadata", {}).get("run_id")
                    == self.checkpoint_metadata.get("run_id", MISSING)
                ),
            )
        )
        updated_channels: set[str] | None = None
```

恢复状态的判断条件为：

  * 检查点存在通道版本（说明确实有中断）
  * 并且配置中 `resuming` 为 True 或逻辑判断为恢复状态：

    * 输入为 None（说明可能是主图首次运行或重启）
    * 输入是 `Command`（说明可能是子图恢复）
    * 或者在非嵌套图中，`run_id` 一致（说明是上次运行）


#### **处理输入为 `Command` 的情况**

➡️ **作用**：将 command 中显式指定的写入应用到状态中。
```python
        # map command to writes
        if isinstance(self.input, Command):
            # 判断 Command resume 保存的是不是一个 map
            if resume_is_map := (
                (resume := self.input.resume) is not None
                and isinstance(resume, dict)
                # 判断 Command.resume 的 key 是否符合中断 ID 的格式
                and all(is_xxh3_128_hexdigest(k) for k in resume)
            ):
                # 将 Command.resume 保存到 config 中
                self.config[CONF][CONFIG_KEY_RESUME_MAP] = self.input.resume
            # 若提供 resume 必须配置 `checkpointer`
            if resume is not None and not self.checkpointer:
                raise RuntimeError(
                    "Cannot use Command(resume=...) without checkpointer"
                )
            writes: defaultdict[str, list[tuple[str, Any]]] = defaultdict(list)
            # group writes by task ID
            # 使用 `map_command(self.input)` 保存在 command 中的，需要执行的 channel 值更新任务
            for tid, c, v in map_command(cmd=self.input):
                if not (c == RESUME and resume_is_map):
                    writes[tid].append((c, v))
            if not writes and not resume_is_map:
                raise EmptyInputError("Received empty Command input")
            # save writes
            for tid, ws in writes.items():
                # 保存这些写入到 task 对应的缓存中（`self.put_writes`）
                self.put_writes(tid, ws)


@dataclass(**_DC_KWARGS)
class Command(Generic[N], ToolOutputMixin):
    graph: str | None = None
    update: Any | None = None
    resume: dict[str, Any] | Any | None = None
    goto: Send | Sequence[Send | N] | N = ()


def map_command(cmd: Command) -> Iterator[tuple[str, str, Any]]:
    """Map input chunk to a sequence of pending writes in the form (channel, value)."""
    if cmd.graph == Command.PARENT:
        raise InvalidUpdateError("There is no parent graph")
    if cmd.goto:
        if isinstance(cmd.goto, (tuple, list)):
            sends = cmd.goto
        else:
            sends = [cmd.goto]
        for send in sends:
            if isinstance(send, Send):
                yield (NULL_TASK_ID, TASKS, send)
            elif isinstance(send, str):
                yield (NULL_TASK_ID, f"branch:to:{send}", START)
            else:
                raise TypeError(
                    f"In Command.goto, expected Send/str, got {type(send).__name__}"
                )
    if cmd.resume is not None:
        yield (NULL_TASK_ID, RESUME, cmd.resume)
    if cmd.update:
        for k, v in cmd._update_as_tuples():
            yield (NULL_TASK_ID, k, v)
```

`self.put_writes` 我们后面详述。


#### **应用 null writes**
➡️ **作用**：恢复一些全局层面（非 task 级别）的写入。

`NULL_TASK_ID` 表示“**当前不存在任务（Task）**”或“**这是一个无任务 ID 的占位符**”。

* 在图的执行过程中，LangGraph 使用 `task_id` 来标识每一个任务节点（node）或 loop 中的迭代任务。
* 但是有些情况下，例如：

  * **初始化阶段**（未开始执行时）
  * **跳过某些节点或任务**
  * **中断或终止**
  * **空的分支**

都可能需要用一个合法但“无含义”的 ID 来表示当前没有实际任务，此时就会用 `NULL_TASK_ID`。

```python
        # apply NULL writes
        # 从待写入列表中找出 `NULL_TASK_ID` 对应的写入（表示不属于任何 task 的全局写入）
        if null_writes := [
            w[1:] for w in self.checkpoint_pending_writes if w[0] == NULL_TASK_ID
        ]:
            # 将这些全局写入应用到当前通道状态中
            apply_writes(
                self.checkpoint,
                self.channels,
                [PregelTaskWrites((), INPUT, null_writes, [])],
                self.checkpointer_get_next_version,
                self.trigger_to_nodes,
            )
```

#### **如果是恢复运行，直接跳到恢复流程**
➡️ **作用**：跳过输入处理，直接进入恢复后的调度。

```python
if is_resuming:
    self.checkpoint["versions_seen"].setdefault(INTERRUPT, {})
    # 设置 seen 版本，用于跳过重复写入
    for k in self.channels:
        if k in self.checkpoint["channel_versions"]:
            # 记录中断恢复时，看到的 channel version
            # 并告诉调度器这些通道是来自中断恢复的
            self.checkpoint["versions_seen"][INTERRUPT][k] = version
    # 调用 `_emit(...)` 发出当前通道值，推动下游执行
    self._emit("values", map_output_values, self.output_keys, True, self.channels)
```

self._emit 方法我们后面再详述。

#### **否则处理正常输入写入流程**
➡️ **作用**：标准运行流程，处理新一轮输入。
```python
        # 使用 `map_input(...)` 从输入中提取写入值，从输入中提取的值限定在 input_channel 内。
        # input_keys 是 loop 初始化时从 Pregel 接收的 input_channels
        elif input_writes := deque(map_input(input_keys, self.input)):
            # discard any unfinished tasks from previous checkpoint
            # 清理上一次运行中未完成的任务（`prepare_next_tasks(...)`）
            # 就是处理 checkpoint_pending_writes
            discard_tasks = prepare_next_tasks(
                self.checkpoint,
                self.checkpoint_pending_writes,
                self.nodes,
                self.channels,
                self.managed,
                self.config,
                self.step,
                self.stop,
                for_execution=True,
                store=None,
                checkpointer=None,
                manager=None,
            )
            # apply input writes
            updated_channels = apply_writes(
                self.checkpoint,
                self.channels,
                [
                    # 处理未完成的写入，
                    *discard_tasks.values(),
                    # 处理新的输入
                    PregelTaskWrites((), INPUT, input_writes, []),
                ],
                self.checkpointer_get_next_version,
                self.trigger_to_nodes,
            )
            # save input checkpoint
            # 更新检查点（标注来源为 input）
            self._put_checkpoint({"source": "input"})
        # 若无输入也不在恢复，抛异常
        elif CONFIG_KEY_RESUMING not in configurable:
            raise EmptyInputError(f"Received no input for {input_keys}")
```

这里比较难理解的部分是 discard_tasks。处理新一轮输入，不一定是这条图有史以来第一次运行：
1. 可能上一次运行一半被 kill、失败或中断；
2. 系统重启或调度器崩溃了；
3. Checkpoint 已保存，但部分任务写入未应用；

传给 apply_writes 有两个部分
1. discard_tasks: 其 `writes=dequeue`，discard_tasks 还没有被执行，所以 writes 为空
2. PregelTaskWrites: 其 `writes=deque(map_input(input_keys, self.input))`

这样经过 apply_writes，未被处理的 channel 就会直接被 update 为空:

```python
    if bump_step:
        for chan in channels:
            if channels[chan].is_available() and chan not in updated_channels:
                if channels[chan].update(EMPTY_SEQ) and next_version is not None:
                    checkpoint["channel_versions"][chan] = next_version
                    # unavailable channels can't trigger tasks, so don't add them
                    if channels[chan].is_available():
                        updated_channels.add(chan)
```

**`channels[TASKS]`是如何恢复的**
这里还有个问题需要注意，prepare_next_tasks 在生成 PUSH 任务时，会先做如下判断:

```python
def prepare_next_tasks():
    tasks_channel = cast(Optional[Topic[Send]], channels.get(TASKS))
    if tasks_channel and tasks_channel.is_available():
```

`channels[TASKS]` 的值是在 Pregel 初始化时添加的，此时 

```python
class Pregel:
    def __init__()
        self.channels = channels or {}
        if TASKS in self.channels and not isinstance(self.channels[TASKS], Topic):
            raise ValueError(
                f"Channel '{TASKS}' is reserved and cannot be used in the graph."
            )
        else:
            self.channels[TASKS] = Topic(Send, accumulate=False)
```

在 Loop `__enter__` 方法中，会调用 channels_from_checkpoint，尝试从 checkpoint 恢复 channel，如果 checkpoint 中存在 `channels[TASKS]` 就会恢复，此时 `channels[TASKS]` 就有值。

最后 `_put_checkpoint` 方法，我们后面再详述。

#### **更新配置、状态**

```python
if not self.is_nested:
    self.config = patch_configurable(self.config, {CONFIG_KEY_RESUMING: is_resuming})
self.status = "pending"
```

* 将 `resuming` 信息注入配置
* 设置当前节点状态为 `"pending"`，准备调度

---

#### ✅ 返回
返回的是 `apply_writes(...)` 中得到的 **被写入的通道名集合**（`set[str]`），用于触发下游节点。


#### 🧠 总结

`_first` 是 Pregel 图首次调度（或恢复）的核心入口：

* 如果是 `Command`，执行恢复流程
* 如果是中断恢复，跳过输入处理
* 如果是正常输入，解析写入并更新通道
* 若都不是，抛出异常
* 最终返回写入了哪些通道，供调度器判断哪些节点应被触发

## 5. after_tick
与 tick 方法对应的还有一个 after_tick 方法，代码不长我们直接来看代码:

```python
    def after_tick(self) -> None:
        # self.tasks 是 tick() 调用 prepare_next_tasks 获取的 Task
        # 获取 Task 的执行结果
        writes = [w for t in self.tasks.values() for w in t.writes]
        # all tasks have finished
        # 应用写入，更新通道，获取被更新的 channel
        self.updated_channels = apply_writes(
            self.checkpoint,
            self.channels,
            self.tasks.values(),
            self.checkpointer_get_next_version,
            self.trigger_to_nodes,
        )
        # produce values output
        # 如果被更新的 channel 和要输出的 channel 存在交集，调用 _emit 方法输出值
        if not self.updated_channels.isdisjoint(
            (self.output_keys,)
            if isinstance(self.output_keys, str)
            else self.output_keys
        ):
            self._emit(
                "values", map_output_values, self.output_keys, writes, self.channels
            )
        # clear pending writes
        # 清空 pending writes
        self.checkpoint_pending_writes.clear()
        # "not skip_done_tasks" only applies to first tick after resuming
        self.skip_done_tasks = True
        # save checkpoint
        # 保存 checkpoint
        self._put_checkpoint({"source": "loop"})
        # after execution, check if we should interrupt
        # 判断是否应该发生中断
        if self.interrupt_after and should_interrupt(
            self.checkpoint, self.interrupt_after, self.tasks.values()
        ):
            self.status = "interrupt_after"
            raise GraphInterrupt()
        # unset resuming flag
        self.config[CONF].pop(CONFIG_KEY_RESUMING, None)
```
