# langgraph Pregel - 2



## 1. Pregel 的方法
前面我们总结过 Pregel 有如下方法。

| 方法名 | 输入类型 | 输出类型 | 作用 |
|--------|----------|----------|------|
| `__init__` | `nodes`, `channels`, `input_channels`, `output_channels` 等 | `None` | 初始化 Pregel 实例，设置节点、通道、输入输出等配置 |
| `get_graph` | `config: RunnableConfig \| None`, `xray: int \| bool = False` | `Graph` | 返回计算图的可绘制表示 |
| `aget_graph` | `config: RunnableConfig \| None`, `xray: int \| bool = False` | `Graph` | 异步返回计算图的可绘制表示 |
| `_repr_mimebundle_` | `**kwargs: Any` | `dict[str, Any]` | Jupyter 显示图形用的 MIME 包 |
| `copy` | `update: dict[str, Any] \| None = None` | `Self` | 创建 Pregel 对象的副本 |
| `with_config` | `config: RunnableConfig \| None = None`, `**kwargs: Any` | `Self` | 使用更新配置创建 Pregel 副本 |
| `validate` | 无参数 | `Self` | 验证图形配置的有效性 |
| `config_schema` | `include: Sequence[str] \| None = None` | `type[BaseModel]` | 获取配置模式（已弃用） |
| `get_config_jsonschema` | `include: Sequence[str] \| None = None` | `dict[str, Any]` | 获取配置 JSON 模式（已弃用） |
| `get_context_jsonschema` | 无参数 | `dict[str, Any] \| None` | 获取上下文 JSON 模式 |
| `get_input_schema` | `config: RunnableConfig \| None = None` | `type[BaseModel]` | 获取输入模式 |
| `get_input_jsonschema` | `config: RunnableConfig \| None = None` | `dict[str, Any]` | 获取输入 JSON 模式 |
| `get_output_schema` | `config: RunnableConfig \| None = None` | `type[BaseModel]` | 获取输出模式 |
| `get_output_jsonschema` | `config: RunnableConfig \| None = None` | `dict[str, Any]` | 获取输出 JSON 模式 |
| `get_subgraphs` | `namespace: str \| None = None`, `recurse: bool = False` | `Iterator[tuple[str, PregelProtocol]]` | 获取图的子图 |
| `aget_subgraphs` | `namespace: str \| None = None`, `recurse: bool = False` | `AsyncIterator[tuple[str, PregelProtocol]]` | 异步获取图的子图 |
| `get_state` | `config: RunnableConfig`, `subgraphs: bool = False` | `StateSnapshot` | 获取图的当前状态 |
| `aget_state` | `config: RunnableConfig`, `subgraphs: bool = False` | `StateSnapshot` | 异步获取图的当前状态 |
| `get_state_history` | `config: RunnableConfig`, `filter`, `before`, `limit` | `Iterator[StateSnapshot]` | 获取图状态历史 |
| `aget_state_history` | `config: RunnableConfig`, `filter`, `before`, `limit` | `AsyncIterator[StateSnapshot]` | 异步获取图状态历史 |
| `bulk_update_state` | `config: RunnableConfig`, `supersteps: Sequence[Sequence[StateUpdate]]` | `RunnableConfig` | 批量更新图状态 |
| `abulk_update_state` | `config: RunnableConfig`, `supersteps: Sequence[Sequence[StateUpdate]]` | `RunnableConfig` | 异步批量更新图状态 |
| `update_state` | `config: RunnableConfig`, `values`, `as_node`, `task_id` | `RunnableConfig` | 更新图状态 |
| `aupdate_state` | `config: RunnableConfig`, `values`, `as_node`, `task_id` | `RunnableConfig` | 异步更新图状态 |
| `stream` | `input`, `config`, `context`, `stream_mode` 等 | `Iterator[dict[str, Any] \| Any]` | 流式执行图并返回步骤输出 |
| `astream` | `input`, `config`, `context`, `stream_mode` 等 | `AsyncIterator[dict[str, Any] \| Any]` | 异步流式执行图并返回步骤输出 |
| `invoke` | `input`, `config`, `context`, `stream_mode` 等 | `dict[str, Any] \| Any` | 同步执行图并返回最终结果 |
| `ainvoke` | `input`, `config`, `context`, `stream_mode` 等 | `dict[str, Any] \| Any` | 异步执行图并返回最终结果 |
| `clear_cache` | `nodes: Sequence[str] \| None = None` | `None` | 清除指定节点的缓存 |
| `aclear_cache` | `nodes: Sequence[str] \| None = None` | `None` | 异步清除指定节点的缓存 |

主要功能分类:

1. **图形管理**: `get_graph`, `aget_graph`, `validate`, `copy`, `with_config`
2. **模式获取**: `get_input_schema`, `get_output_schema`, `get_context_jsonschema` 等
3. **状态管理**: `get_state`, `aget_state`, `get_state_history`, `update_state` 等
4. **执行控制**: `invoke`, `ainvoke`, `stream`, `astream`
5. **缓存管理**: `clear_cache`, `aclear_cache`
6. **子图操作**: `get_subgraphs`, `aget_subgraphs`

上一节我们介绍了 invoke 方法的入参，现在我们就从 invoke 方法的实现开始。

## 2. invoke 方法
invoke 的核心调用的是 stream 方法。并根据不同的 stream_model 返回不同的输出。

```python
    def invoke(
        self,
        input: InputT | Command | None,
        config: RunnableConfig | None = None,
        *,
        context: ContextT | None = None,
        stream_mode: StreamMode = "values",
        print_mode: StreamMode | Sequence[StreamMode] = (),
        output_keys: str | Sequence[str] | None = None,
        interrupt_before: All | Sequence[str] | None = None,
        interrupt_after: All | Sequence[str] | None = None,
        **kwargs: Any,
    ) -> dict[str, Any] | Any:
        # 控制输出哪些 channel 
        output_keys = output_keys if output_keys is not None else self.output_channels

        latest: dict[str, Any] | Any = None
        chunks: list[dict[str, Any] | Any] = []
        interrupts: list[Interrupt] = []

        for chunk in self.stream(
            input,
            config,
            context=context,
            stream_mode=["updates", "values"]
            if stream_mode == "values"
            else stream_mode,
            print_mode=print_mode,
            output_keys=output_keys,
            interrupt_before=interrupt_before,
            interrupt_after=interrupt_after,
            **kwargs,
        ):
            if stream_mode == "values":
                if len(chunk) == 2:
                    mode, payload = cast(tuple[StreamMode, Any], chunk)
                else:
                    _, mode, payload = cast(
                        tuple[tuple[str, ...], StreamMode, Any], chunk
                    )
                if (
                    mode == "updates"
                    and isinstance(payload, dict)
                    and (ints := payload.get(INTERRUPT)) is not None
                ):
                    interrupts.extend(ints)
                elif mode == "values":
                    latest = payload
            else:
                chunks.append(chunk)

        if stream_mode == "values":
            if interrupts:
                return (
                    {**latest, INTERRUPT: interrupts}
                    if isinstance(latest, dict)
                    else {INTERRUPT: interrupts}
                )
            return latest
        else:
            return chunks
```
