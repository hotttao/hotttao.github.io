# pregel channel 读写


前面我们介绍了 langgraph 中的 channel 类型，这一节我们来介绍 channel 的读写。

## 1. ChannelRead

```bash
提问: 解释一下  ChannelRead 的语义，并以表格列举 ChannelRead 的属性，以另一个表格列出每个方法名、作用、输出值类型
```

ChannelRead 定义在 `langgraph\pregel\_read.py` 与 PregelNode 位于同一个 py 文件。`ChannelRead` 是 LangGraph 中的一个工具类，主要用于：

> **从当前执行上下文（`RunnableConfig`）中读取某个 channel 的 state 值，用于在图节点中访问数据。**

它既可以作为 LCEL 的一个 `Runnable` 使用，也可以直接调用其 `do_read()` 静态方法来读取值。

---


### 1.1 `ChannelRead` 的属性

```python
class ChannelRead(RunnableCallable):
    """Implements the logic for reading state from CONFIG_KEY_READ.
    Usable both as a runnable as well as a static method to call imperatively."""

    channel: str | list[str]

    fresh: bool = False

    mapper: Callable[[Any], Any] | None = None

    def __init__(
        self,
        channel: str | list[str],
        *,
        fresh: bool = False,
        mapper: Callable[[Any], Any] | None = None,
        tags: list[str] | None = None,
    ) -> None:
        super().__init__(
            func=self._read,
            afunc=self._aread,
            tags=tags,
            name=None,
            trace=False,
        )
        self.fresh = fresh
        self.mapper = mapper
        self.channel = channel
```

| 属性名       | 类型                       | 默认值           | 作用                           |                      |
| --------- | ------------------------ | ------------- | ---------------------------- | -------------------- |
| `channel` | `str \| list[str]`  | 无                            | 要读取的 channel 名称或名称列表 |
| `fresh`   | `bool`                   | `False`       | 是否强制从最新的 checkpoint 读取（跳过缓存） |                      |
| `mapper`  | `Callable[[Any], Any]` | `None`                       | 对读取的结果进行后处理的函数       |

RunnableCallable 是 Langgraph 定义的类似 RunnableLambda。

```python
class RunnableCallable(Runnable):
    """A much simpler version of RunnableLambda that requires sync and async functions."""

    def __init__(
        self,
        func: Callable[..., Any | Runnable] | None,
        afunc: Callable[..., Awaitable[Any | Runnable]] | None = None,
        *,
        name: str | None = None,
        tags: Sequence[str] | None = None,
        trace: bool = True,
        recurse: bool = True,
        explode_args: bool = False,
        **kwargs: Any,
    ) -> None:
        pass
```


### 1.2 `ChannelRead` 的方法说明

| 方法名        | 作用说明                                               | 输出值类型  |
| ---------- | -------------------------------------------------- | ------ |
| `get_name` | 生成 runnable 的可视化名称（用于追踪或调试）                        | `str`  |
| `_read`    | 同步读取指定 channel 的值，从 config\[CONFIG\_KEY\_READ] 中获取 | `Any`  |
| `_aread`   | 异步版本的 `_read`，逻辑一致                                 | `Any`  |
| `do_read`  | 静态方法，实现核心读取逻辑。可独立于实例调用，支持外部自定义调用场景。                | `Any`  |

```python

CONF = cast(Literal["configurable"], sys.intern("configurable"))
CONFIG_KEY_READ = sys.intern("__pregel_read")
READ_TYPE = Callable[[Union[str, Sequence[str]], bool], Union[Any, dict[str, Any]]]

class ChannelRead(RunnableCallable):
    def get_name(self, suffix: str | None = None, *, name: str | None = None) -> str:
        if name:
            pass
        elif isinstance(self.channel, str):
            name = f"ChannelRead<{self.channel}>"
        else:
            name = f"ChannelRead<{','.join(self.channel)}>"
        return super().get_name(suffix, name=name)

    def _read(self, _: Any, config: RunnableConfig) -> Any:
        return self.do_read(
            config, select=self.channel, fresh=self.fresh, mapper=self.mapper
        )

    async def _aread(self, _: Any, config: RunnableConfig) -> Any:
        return self.do_read(
            config, select=self.channel, fresh=self.fresh, mapper=self.mapper
        )

    @staticmethod
    def do_read(
        config: RunnableConfig,
        *,
        select: str | list[str], # channel 的名称
        fresh: bool = False,     
        mapper: Callable[[Any], Any] | None = None,
    ) -> Any:
        try:
            # 从 configurable 的 __pregel_read 获取调用读取 channel 的函数
            read: READ_TYPE = config[CONF][CONFIG_KEY_READ]
        except KeyError:
            raise RuntimeError(
                "Not configured with a read function"
                "Make sure to call in the context of a Pregel process"
            )
        if mapper:
            return mapper(read(select, fresh))
        else:
            return read(select, fresh)

```

#### CONF
CONF = cast(Literal["configurable"], sys.intern("configurable"))

| 部分                                   | 含义                                                              |
| ------------------------------------ | --------------------------------------------------------------- |
| `sys.intern("configurable")`         | 将字符串 `"configurable"` 放入 Python 的内部字符串池中，确保所有值相等的字符串共享内存（性能优化）。 |
| `cast(Literal["configurable"], ...)` | 让类型检查器（如 mypy）知道这个值的类型是 `Literal["configurable"]`（一个固定的字面值）。    |
| `CONF = ...`                         | 给变量 `CONF` 赋值为 `"configurable"`，并且类型是 `Literal["configurable"]` |

#### do_read
do_read 中 select 和 fresh 是读取 channel 值时的核心控制参数，用于控制读取哪个 channel、是否读取最新值（跳过 cache）。

## 2. ChannelWrite

```bash
提问: 解释一下  ChannelWrite 的语义，并以表格列举 ChannelRead 的属性，以另一个表格列出每个方法名、作用、输出值类型
```

ChannelWrite 负责将值写入指定的 channel，它是 LangGraph 中的 “输出指令器”

* 写入中间状态：将模型输出写入 `EphemeralValue` 等 channel。
* 跨步骤传递值：将当前步骤的产出传递给下一个节点读取。
* 自动支持静态分析：可在编译时收集写入信息，便于优化。

### 2.1 ChannelWrite 属性
ChannelWrite 只有一个 `writes: list[ChannelWriteEntry | ChannelWriteTupleEntry | Send]` 属性。表示要写入的内容。

```python
class ChannelWrite(RunnableCallable):
    """Implements the logic for sending writes to CONFIG_KEY_SEND.
    Can be used as a runnable or as a static method to call imperatively."""

    writes: list[ChannelWriteEntry | ChannelWriteTupleEntry | Send]
    """Sequence of write entries or Send objects to write."""

    def __init__(
        self,
        writes: Sequence[ChannelWriteEntry | ChannelWriteTupleEntry | Send],
        *,
        tags: Sequence[str] | None = None,
    ):
        super().__init__(
            func=self._write,
            afunc=self._awrite,
            name=None,
            tags=tags,
            trace=False,
        )
        self.writes = cast(
            list[Union[ChannelWriteEntry, ChannelWriteTupleEntry, Send]], writes
        )
```

我们先来 writes 的三种写入类型。

| 特性           | `ChannelWriteEntry` | `ChannelWriteTupleEntry`         |
| ------------ | ------------------- | -------------------------------- |
| 🌟 语义        | 写入一个指定的 channel     | 从输入中提取多个 (channel, value) 组合进行写入 |
| 📌 channel 名 | 静态字符串，一个            | 动态，从 mapper 返回的 tuple 中提取        |
| 🎯 使用 mapper | 可选：用于修改 `value`     | 必须：从输入生成写入列表                     |
| 🎯 写入数量      | 通常是一个 channel       | 通常是多个 channel                    |
| 🔍 static 用法 | 不常用（默认写入是固定的）       | 通常用于声明所有可能写入的 channel，供静态分析用     |


#### ChannelWriteEntry

```python
class ChannelWriteEntry(NamedTuple):
    channel: str
    """Channel name to write to."""
    value: Any = PASSTHROUGH
    """Value to write, or PASSTHROUGH to use the input."""
    skip_none: bool = False
    """Whether to skip writing if the value is None."""
    mapper: Callable | None = None
    """Function to transform the value before writing."""
```


| 属性名         | 类型                     | 说明                                      |                                            |
| ----------- | ---------------------- | --------------------------------------- | ------------------------------------------ |
| `channel`   | `str`                  | 要写入的目标 channel 名称。                      |                                            |
| `value`     | `Any`，默认 `PASSTHROUGH` | 要写入的值。如果是 `PASSTHROUGH`，则表示使用输入数据作为写入值。 |                                            |
| `skip_none` | `bool`，默认 `False`      | 如果为 `True` 且 `value is None`，则跳过本次写入。   |                                            |
| `mapper`    | `Callable\| None`  | 可选函数，用于对 `value` 做变换后再写入（接收 `value` 作为参数）。 |

#### ChannelWriteTupleEntry

```python
class ChannelWriteTupleEntry(NamedTuple):
    mapper: Callable[[Any], Sequence[tuple[str, Any]] | None]
    """Function to extract tuples from value."""
    value: Any = PASSTHROUGH
    """Value to write, or PASSTHROUGH to use the input."""
    static: Sequence[tuple[str, Any, str | None]] | None = None
    """Optional, declared writes for static analysis."""
```

| 属性名      | 说明                                                     |
| -------- | ------------------------------------------------------ |
| `mapper` | 一个函数，用于从输入值中提取 `(channel, value)` 元组的序列。               |
| `value`  | 要传入 `mapper` 的值；若为 `PASSTHROUGH`，表示使用外部输入值。            |
| `static` | 可选的静态写入声明，供静态分析使用，格式为 `(channel, value, label)` 的元组序列。 |


static 字段在 LangGraph 中扮演了一个用于「静态分析」写入操作的声明性机制。其是一个包含多个三元组的序列

| 元素             | 含义                       |                   |
| -------------- | ------------------------ | ----------------- |
| `channel: str` | 要写入的通道名称                 |                   |
| `value: Any`   | 要写入的值（通常是占位符、代表类型、或预估结构） |                   |
| `label: str \| None`                   | 可选标签，用于调试、追踪或图可视化 |


LangGraph 在 构图 或 编译阶段 使用 static 信息来：
1. 提前知道一个节点将写哪些 channel
2. 构建数据依赖图


#### Send
`Send` 是 LangGraph 中用于**动态调度特定节点**的一种机制。它的语义可以总结为：

> **“携带一个子状态，定向投递给某个指定节点执行。”**

背景语义:

在普通的流程图（graph）执行中，状态在节点之间按顺序流动。但有些场景下，你希望：

* 并行地将不同的状态发给同一个节点（例如 map-reduce 中的 map 阶段），
* 或者跳过主状态流转，直接调用某个子图或分支。

这时就可以使用 `Send` 对象，它允许你在运行时“手动”指定：

* **发送给哪个节点（`node`）**
* **发送什么状态（`arg`）**

这允许 LangGraph 实现非常灵活的状态调度逻辑。
* 可以把 `Send(node="X", arg={...})` 理解为：“下一步，请执行节点 X，输入状态是 {...}，不要用当前全局状态。”
* 类似于“**有条件跳转 + 局部状态替换**”。

---

```python
class Send:
    __slots__ = ("node", "arg")

    node: str
    arg: Any

    def __init__(self, /, node: str, arg: Any) -> None:
        """
        Initialize a new instance of the Send class.

        Args:
            node: The name of the target node to send the message to.
            arg: The state or message to send to the target node.
        """
        self.node = node
        self.arg = arg
```

| 属性名    | 说明                       |
| ------ | ------------------------ |
| `node` | 要发送状态的目标节点名称（字符串）        |
| `arg`  | 要发送的状态（可为任何对象，通常是部分状态字典） |

Send 没有具体的方法，只是一个数据装载的容器。

### 2.2 ChannelWrite 方法

ChannelWrite 有如下一些方法:

| 方法名                        | 作用描述                                                 | 输出类型                         |                  |
| -------------------------- | ---------------------------------------------------- | ---------------------------- | ---------------- |
| `get_name`                 | 自动生成节点名称（如 `ChannelWrite<input>`）用于图调试               | `str`                        |                  |
| `_write(input, config)`    | 同步写入逻辑，将 input 写入 channel，支持 `PASSTHROUGH` 替换        | `Any`（传回 input）              |                  |
| `_awrite(input, config)`   | 异步版本的写入逻辑                                            | `Any`（传回 input）              |                  |
| `do_write(config, writes)` | 静态方法，真正执行写入逻辑，调用配置中的 `send` 函数                       | `None`                       |                  |
| `is_writer(runnable)`      | 判断一个 runnable 是否是 writer（用于 `PregelNode` 识别）         | `bool`                       |                  |
| `get_static_writes()`      | 获取 static 写入声明（用于静态分析、优化）                            | \`list\[tuple\[str, Any, str | None]] \| None\` |
| `register_writer()`        | 手动注册非 `ChannelWrite` 的 runnable 为 writer，并可声明其静态写入行为 | `R`（泛型）                      |                  |


### 4.1 _write
_write 调用的是  do_write 方法，在调用前将 PASSTHROUGH 替换为 input。

```python
PASSTHROUGH = object()

class ChannelWrite(RunnableCallable):
    def _write(self, input: Any, config: RunnableConfig) -> None:
        writes = [
            (
                # 将 PASSTHROUGH 替换为 input
                ChannelWriteEntry(write.channel, input, write.skip_none, write.mapper)
                if isinstance(write, ChannelWriteEntry) and write.value is PASSTHROUGH
                else (
                    ChannelWriteTupleEntry(write.mapper, input)
                    if isinstance(write, ChannelWriteTupleEntry)
                    and write.value is PASSTHROUGH
                    else write
                )
            )
            for write in self.writes
        ]
        self.do_write(
            config,
            writes,
        )
        return input
```

### 4.2 do_write

```python
TASKS = sys.intern("__pregel_tasks")
CONF = cast(Literal["configurable"], sys.intern("configurable"))
CONFIG_KEY_SEND = sys.intern("__pregel_send")

class ChannelWrite(RunnableCallable):
    @staticmethod
    def do_write(
        config: RunnableConfig,
        writes: Sequence[ChannelWriteEntry | ChannelWriteTupleEntry | Send],
        allow_passthrough: bool = True,
    ) -> None:
        # validate
        for w in writes:
            if isinstance(w, ChannelWriteEntry):
                # 检查是否为 TASKS 通道
                if w.channel == TASKS:
                    raise InvalidUpdateError(
                        "Cannot write to the reserved channel TASKS"
                    )
                if w.value is PASSTHROUGH and not allow_passthrough:
                    raise InvalidUpdateError("PASSTHROUGH value must be replaced")
            if isinstance(w, ChannelWriteTupleEntry):
                if w.value is PASSTHROUGH and not allow_passthrough:
                    raise InvalidUpdateError("PASSTHROUGH value must be replaced")
        # if we want to persist writes found before hitting a ParentCommand
        # can move this to a finally block
        # 从 configurable 的 __pregel_send 获取调用往 channel 写入值的函数
        write: TYPE_SEND = config[CONF][CONFIG_KEY_SEND]
        write(_assemble_writes(writes))

# 计算要写入的值返回 (channel, value) 元组列表
def _assemble_writes(
    writes: Sequence[ChannelWriteEntry | ChannelWriteTupleEntry | Send],
) -> list[tuple[str, Any]]:
    """Assembles the writes into a list of tuples."""
    tuples: list[tuple[str, Any]] = []
    for w in writes:
        if isinstance(w, Send):
            tuples.append((TASKS, w))
        elif isinstance(w, ChannelWriteTupleEntry):
            if ww := w.mapper(w.value):
                tuples.extend(ww)
        elif isinstance(w, ChannelWriteEntry):
            value = w.mapper(w.value) if w.mapper is not None else w.value
            if value is SKIP_WRITE:
                continue
            if w.skip_none and value is None:
                continue
            tuples.append((w.channel, value))
        else:
            raise ValueError(f"Invalid write entry: {w}")
    return tuples
```

### 4.3 writer 管理
ChannelWrite 下的三个方法与 writer 的管理有关:

```python
    @staticmethod
    def is_writer(runnable: Runnable) -> bool:
        """Used by PregelNode to distinguish between writers and other runnables."""
        return (
            isinstance(runnable, ChannelWrite)
            or getattr(runnable, "_is_channel_writer", MISSING) is not MISSING
        )

    @staticmethod
    def get_static_writes(
        runnable: Runnable,
    ) -> Sequence[tuple[str, Any, str | None]] | None:
        """Used to get conditional writes a writer declares for static analysis."""
        if isinstance(runnable, ChannelWrite):
            return [
                w
                for entry in runnable.writes
                if isinstance(entry, ChannelWriteTupleEntry) and entry.static
                for w in entry.static
            ] or None
        elif writes := getattr(runnable, "_is_channel_writer", MISSING):
            if writes is not MISSING:
                writes = cast(
                    Sequence[tuple[Union[ChannelWriteEntry, Send], Optional[str]]],
                    writes,
                )
                entries = [e for e, _ in writes]
                labels = [la for _, la in writes]
                return [(*t, la) for t, la in zip(_assemble_writes(entries), labels)]

    @staticmethod
    def register_writer(
        runnable: R,
        static: Sequence[tuple[ChannelWriteEntry | Send, str | None]] | None = None,
    ) -> R:
        """Used to mark a runnable as a writer, so that it can be detected by is_writer.
        Instances of ChannelWrite are automatically marked as writers.
        Optionally, a list of declared writes can be passed for static analysis."""
        # using object.__setattr__ to work around objects that override __setattr__
        # eg. pydantic models and dataclasses
        object.__setattr__(runnable, "_is_channel_writer", static)
        return runnable
```

这些方法的关系如下:
1. register_writer: 
    - 标记一个 runnable 为“写操作节点”。方法是设置 `_is_channel_writer` 属性。
    - 可以添加 static 信息 (channel， Value) 的元组序列。
2. is_writer: 通过检查 `_is_channel_writer` 属性判断是否是 writer
3. get_static_writes: 
    - 提取 writer 的 static 信息
    - 反向解析 register_writer 添加的 static 信息

## 5. 总结
channel 的读取和写入最终调用的是 RunnableConfig 中配置的读取和写入函数:

config["configurable"]
    - __pregel_read: 读取函数的索引 key
    - __pregel_send: 发送函数的索引 key

