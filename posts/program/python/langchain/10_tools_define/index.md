# LangChain Tools Define


我们在前面介绍了 Tool 相关的 Message 传递以及解析，这一节我们来看 Langchain 对 Tool 本身的抽象。

## 2. Tool 的 UML 类图
![Tool 类图](/images/langchain/tool.svg)

如图，OutPut 的代表比较简单，有如下类:
1. BaseTool
2. Tool
3. StructuredTool
4. BaseToolkit
5. InjectableToolArg
6. InjectableToolCallId
6. RetrieverInput
7. _SchemaConfig

### 2.1 提问
Tool 中的内不多，让我们直接问大模型这些类的语义

```bash
我正在阅读  langchain tool 子包的源代码，注意到包内，有如下类:
1. BaseTool
2. Tool
3. StructuredTool
4. BaseToolkit
5. InjectableToolArg
6. InjectableToolCallId
6. RetrieverInput
7. _SchemaConfig
请给我从语义上解释一下这些类的的作用

```

你正在研究 `langchain-core` 中的 `langchain_core/tools`（或 `langchain/tools`）子包。这是用于支持 **Tool Calling** 的核心模块，封装了从基本工具定义、参数解析，到支持 structured input/output 的整个工具系统。

下面我从**语义角度**为你解释这些类的作用和定位。

---


#### 1. `BaseTool`

**语义**：所有 Tool 的抽象基类。

**作用**：

* 定义工具的通用接口：如 `name`, `description`, `args_schema`, `invoke`, `run` 等；
* 支持同步和异步调用；
* 与 LangChain 的 `Runnable` 接口集成；
* 主要作为 `Tool` 和 `StructuredTool` 的父类存在。

---

#### 2. `Tool`

**语义**：**最简单的 Tool 实现**，基于函数封装的工具。

**作用**：

* 接收一个 Python 函数和描述；
* 自动构造 Tool 的输入模式（仅支持 `str -> str`）；
* 比如：

```python
def get_time(location: str) -> str:
    ...

tool = Tool.from_function(get_time, name="GetTime", description="获取时间")
```

适合**无结构参数**的简单函数封装。

---

#### 3. `StructuredTool`

**语义**：**结构化输入**的 Tool 实现，基于 `Pydantic` 的 schema 校验。

**作用**：

* 支持复杂输入结构，如多个参数、类型校验、自动 OpenAI-compatible schema；
* 自动将 `Pydantic` 输入模型绑定到 Python 函数；
* 用于 **OpenAI Tool Calling** 和 **LangChain Function Calling**；

示例：

```python
class GetWeatherInput(BaseModel):
    city: str
    unit: Literal["celsius", "fahrenheit"]

def get_weather(city: str, unit: str) -> str:
    ...

tool = StructuredTool.from_function(
    func=get_weather,
    args_schema=GetWeatherInput,
    name="get_weather"
)
```

---


#### 4. `BaseToolkit`

**语义**：多个工具的组合体，提供动态注入或注册能力。

**作用**：

* 代表一组相关的工具（如文件处理工具、数据库工具等）；
* 常用于 agent 中批量注册；
* 通常提供 `.get_tools()` 方法，返回多个 Tool 实例。

例如：

```python
toolkit = FileToolkit()
agent = initialize_agent(tools=toolkit.get_tools(), ...)
```

---

#### 5. `InjectableToolArg`

**语义**：标记“可注入参数”的类型，用于支持上下文绑定（如用户 ID、调用上下文等）。

**作用**：

* 可通过上下文自动注入到 Tool 的输入参数中；
* 常见于使用 `Runnable` 或 LangChain Context 时；

例子：

```python
def some_tool(user_id: Annotated[str, InjectableToolArg]) -> str:
    return f"Hello {user_id}"
```

---

#### 6. `InjectableToolCallId`

**语义**：特化版本的 `InjectableToolArg`，专门表示“Tool 被调用的唯一 ID”。

**作用**：

* 用于标记该字段应由 LangChain 自动生成；
* 方便 Tool 执行日志追踪、链路上下文管理；

例子：

```python
def tool_func(call_id: Annotated[str, InjectableToolCallId]):
    ...
```

---

#### 7. `RetrieverInput`

**语义**：为 `RetrieverTool` 提供的结构化输入模型。

**作用**：

* 表示 Retriever 类型工具的输入参数（如 search query）；
* 是一个标准的 `BaseModel` 子类；
* 一般含字段如 `query: str`

---

#### 8. `_SchemaConfig`

**语义**：为 `StructuredTool` 自动生成 JSON Schema 所用的内部配置类。

**作用**：

* 控制 Pydantic 模型序列化行为，如禁用额外字段、字段顺序；
* 一般通过继承注入到工具的 `args_schema` 中；
* 内部用途，不建议手动使用。

---

#### 🧭 总结关系图谱（简化）

```
                            +----------------+
                            |   BaseTool     |  <---- 统一接口定义 (Runnable)
                            +--------+-------+
                                     |
              +----------------------+---------------------+
              |                                            |
         +----v-----+                                +-----v-----------+
         |  Tool     |    (非结构化工具封装)          | StructuredTool  | (结构化输入 Tool)
         +----------+                                +-----------------+
              |
         +----v------------------+
         | args: str -> str      |
         +-----------------------+


        +--------------------+             +---------------------------+
        |  BaseToolkit       |--.tools()--> [Tool, Tool, StructuredTool...]
        +--------------------+

        +------------------------+
        | InjectableToolArg      |----+
        | InjectableToolCallId   |    | 用于参数注入
        +------------------------+    |

        +---------------------+
        | RetrieverInput      | <-- 专为检索工具设计的输入模型
        +---------------------+

        +---------------------+
        | _SchemaConfig       | <-- 用于 schema 自动化校验 (Pydantic 配置)
        +---------------------+
```

## 2. BaseTool

BaseTool 是所有 Tool 的抽象基类，定义了工具的通用接口。代码比较长，我们先来看其属性

### 2.1 BaseTool 的属性

```python
class BaseTool(RunnableSerializable[Union[str, dict, ToolCall], Any]):
    """Base class for all LangChain tools.

    This abstract class defines the interface that all LangChain tools must implement.
    Tools are components that can be called by agents to perform specific actions.
    """

    def __init_subclass__(cls, **kwargs: Any) -> None:
        """Validate the tool class definition during subclass creation.

        Args:
            **kwargs: Additional keyword arguments passed to the parent class.

        Raises:
            SchemaAnnotationError: If args_schema has incorrect type annotation.
        """
        super().__init_subclass__(**kwargs)

        args_schema_type = cls.__annotations__.get("args_schema", None)

        if args_schema_type is not None and args_schema_type == BaseModel:
            # Throw errors for common mis-annotations.
            # TODO: Use get_args / get_origin and fully
            # specify valid annotations.
            typehint_mandate = """
class ChildTool(BaseTool):
    ...
    args_schema: Type[BaseModel] = SchemaClass
    ..."""
            name = cls.__name__
            msg = (
                f"Tool definition for {name} must include valid type annotations"
                f" for argument 'args_schema' to behave as expected.\n"
                f"Expected annotation of 'Type[BaseModel]'"
                f" but got '{args_schema_type}'.\n"
                f"Expected class looks like:\n"
                f"{typehint_mandate}"
            )
            raise SchemaAnnotationError(msg)

    name: str
    """The unique name of the tool that clearly communicates its purpose."""
    description: str
    """Used to tell the model how/when/why to use the tool.

    You can provide few-shot examples as a part of the description.
    """

    args_schema: Annotated[Optional[ArgsSchema], SkipValidation()] = Field(
        default=None, description="The tool schema."
    )
    """Pydantic model class to validate and parse the tool's input arguments.

    Args schema should be either:

    - A subclass of pydantic.BaseModel.
    or
    - A subclass of pydantic.v1.BaseModel if accessing v1 namespace in pydantic 2
    or
    - a JSON schema dict
    """
    return_direct: bool = False
    """Whether to return the tool's output directly.

    Setting this to True means
    that after the tool is called, the AgentExecutor will stop looping.
    """
    verbose: bool = False
    """Whether to log the tool's progress."""

    callbacks: Callbacks = Field(default=None, exclude=True)
    """Callbacks to be called during tool execution."""

    callback_manager: Optional[BaseCallbackManager] = deprecated(
        name="callback_manager", since="0.1.7", removal="1.0", alternative="callbacks"
    )(
        Field(
            default=None,
            exclude=True,
            description="Callback manager to add to the run trace.",
        )
    )
    tags: Optional[list[str]] = None
    """Optional list of tags associated with the tool. Defaults to None.
    These tags will be associated with each call to this tool,
    and passed as arguments to the handlers defined in `callbacks`.
    You can use these to eg identify a specific instance of a tool with its use case.
    """
    metadata: Optional[dict[str, Any]] = None
    """Optional metadata associated with the tool. Defaults to None.
    This metadata will be associated with each call to this tool,
    and passed as arguments to the handlers defined in `callbacks`.
    You can use these to eg identify a specific instance of a tool with its use case.
    """

    handle_tool_error: Optional[Union[bool, str, Callable[[ToolException], str]]] = (
        False
    )
    """Handle the content of the ToolException thrown."""

    handle_validation_error: Optional[
        Union[bool, str, Callable[[Union[ValidationError, ValidationErrorV1]], str]]
    ] = False
    """Handle the content of the ValidationError thrown."""

    response_format: Literal["content", "content_and_artifact"] = "content"
    """The tool response format. Defaults to 'content'.

    If "content" then the output of the tool is interpreted as the contents of a
    ToolMessage. If "content_and_artifact" then the output is expected to be a
    two-tuple corresponding to the (content, artifact) of a ToolMessage.
    """

    def __init__(self, **kwargs: Any) -> None:
        """Initialize the tool."""
        if (
            "args_schema" in kwargs
            and kwargs["args_schema"] is not None
            and not is_basemodel_subclass(kwargs["args_schema"])
            and not isinstance(kwargs["args_schema"], dict)
        ):
            msg = (
                "args_schema must be a subclass of pydantic BaseModel or "
                f"a JSON schema dict. Got: {kwargs['args_schema']}."
            )
            raise TypeError(msg)
        super().__init__(**kwargs)

    model_config = ConfigDict(
        arbitrary_types_allowed=True,
    )
```

`BaseTool` 字段:

| 字段名                       | 类型                                                               | 含义              | 作用                                    | 示例值                                    |
| ------------------------- | ---------------------------------------------------------------- | --------------- | ------------------------------------- | -------------------------------------- |
| `name`                    | `str`                                                            | 工具名称（唯一标识）      | 被 agent 用来选择调用哪个工具（必需字段）              | `"get_weather"`                        |
| `description`             | `str`                                                            | 工具描述            | 提示 LLM 何时、为何、如何使用此工具，通常会出现在 prompt 中  | `"Get weather info for a city"`        |
| `args_schema`             | `Optional[ArgsSchema]`（一般是 `Type[BaseModel]` 或 JSON schema dict） | 工具参数的 schema 描述 | 定义工具输入的结构（用于自动解析、验证、OpenAI schema 生成） | `MyArgsSchema`（Pydantic 类）             |
| `return_direct`           | `bool`                                                           | 是否直接返回工具输出      | 如果为 True，调用工具后 Agent 停止执行并直接返回结果      | `True`                                 |
| `verbose`                 | `bool`                                                           | 是否开启调试日志        | 控制是否在运行时输出调试信息                        | `False`                                |
| `callbacks`               | `Callbacks`                                                      | 工具执行时的回调函数集合    | 用于监听工具生命周期中的事件（如 start, end, error）   | `lambda ...: ...`                      |
| `callback_manager`        | `Optional[BaseCallbackManager]`（已废弃）                             | 旧版回调系统          | 被 `callbacks` 替代，仅为兼容老版本保留            | 已废弃                                    |
| `tags`                    | `Optional[list[str]]`                                            | 标签              | 可用于标识工具用途、版本、来源等元信息，在 tracing 中辅助区分   | `["weather", "v1"]`                    |
| `metadata`                | `Optional[dict[str, Any]]`                                       | 元数据             | 附加在每次工具调用上的上下文信息，可被 tracing 使用        | `{"source": "weather_api"}`            |
| `handle_tool_error`       | `bool \| str \| Callable`                                        | 工具执行失败时如何处理异常   | 可以自定义错误提示或屏蔽报错，提升 agent 鲁棒性           | `lambda e: f"Error: {e}"`              |
| `handle_validation_error` | `bool \| str \| Callable`                                        | 参数验证失败时如何处理异常   | 类似上面，但作用于参数校验失败（来自 Pydantic）          | `lambda e: "Invalid input"`            |
| `response_format`         | `Literal["content", "content_and_artifact"]`                     | 工具返回结果的格式       | 用于支持 ToolMessage 附带 artifact（如图片、文件）  | `"content"` 或 `"content_and_artifact"` |

在 `__init__` 构造函数中还做了一项重要校验：

```python
if "args_schema" in kwargs and not is_basemodel_subclass(...) and not isinstance(..., dict):
    raise TypeError(...)
```

> ✅ 确保 `args_schema` 是 `Pydantic` 模型类 或 JSON schema dict，防止开发者错误传了一个实例。

---

类方法 `__init_subclass__` 的作用，当你 **定义一个 BaseTool 的子类** 时，它会自动触发 `__init_subclass__`，做如下检查：

* 如果你这样写：

  ```python
  class MyTool(BaseTool):
      args_schema: BaseModel = MySchema  # ❌ 错误！
  ```

  就会抛出 `SchemaAnnotationError`，提示你应使用：

  ```python
  args_schema: Type[BaseModel] = MySchema  # ✅ 正确
  ```

注:
1. `BaseModel` 表示一个 对象实例，比如 MyModel()；
2. `Type[BaseModel]` 表示一个 类本身，比如 MyModel。

这里对 BaseTool 的属性做一个总结:

| 功能类别    | 相关字段                                           |
| ------- | ---------------------------------------------- |
| Tool 识别 | `name`, `description`                          |
| 输入校验    | `args_schema`, `handle_validation_error`       |
| 输出控制    | `return_direct`, `response_format`             |
| 日志/调试   | `verbose`, `callbacks`, `tags`, `metadata`     |
| 异常处理    | `handle_tool_error`, `handle_validation_error` |
| 向后兼容    | `callback_manager`（已废弃）                        |

### 2.2 BaseTool 的接口
BaseTool 继承了 Runable，他实现了 invoke 接口:
1. invoke 调用了 run 方法
2. run 则调用了 _run 方法
3. _run是 BaseTool 唯一抽象的接口方法。

```python
class BaseTool(RunnableSerializable[Union[str, dict, ToolCall], Any]):
    @override
    def invoke(
        self,
        input: Union[str, dict, ToolCall],
        config: Optional[RunnableConfig] = None,
        **kwargs: Any,
    ) -> Any:
        tool_input, kwargs = _prep_run_args(input, config, **kwargs)
        return self.run(tool_input, **kwargs)

    # --- Tool ---

    def _parse_input(
        self, tool_input: Union[str, dict], tool_call_id: Optional[str]
    ) -> Union[str, dict[str, Any]]:
        """Parse and validate tool input using the args schema.

        Args:
            tool_input: The raw input to the tool.
            tool_call_id: The ID of the tool call, if available.

        Returns:
            The parsed and validated input.

        Raises:
            ValueError: If string input is provided with JSON schema or if
                InjectedToolCallId is required but not provided.
            NotImplementedError: If args_schema is not a supported type.
        """
        pass

    @abstractmethod
    def _run(self, *args: Any, **kwargs: Any) -> Any:
        """Use the tool.

        Add run_manager: Optional[CallbackManagerForToolRun] = None
        to child implementations to enable tracing.
        """

    @deprecated("0.1.47", alternative="invoke", removal="1.0")
    def __call__(self, tool_input: str, callbacks: Callbacks = None) -> str:
        """Make tool callable (deprecated).

        Args:
            tool_input: The input to the tool.
            callbacks: Callbacks to use during execution.

        Returns:
            The tool's output.
        """
        return self.run(tool_input, callbacks=callbacks)
```

### 2.3 run 方法
```python
class BaseTool(RunnableSerializable[Union[str, dict, ToolCall], Any]):
    def run(
        self,
        tool_input: Union[str, dict[str, Any]],
        verbose: Optional[bool] = None,  # noqa: FBT001
        start_color: Optional[str] = "green",
        color: Optional[str] = "green",
        callbacks: Callbacks = None,
        *,
        tags: Optional[list[str]] = None,
        metadata: Optional[dict[str, Any]] = None,
        run_name: Optional[str] = None,
        run_id: Optional[uuid.UUID] = None,
        config: Optional[RunnableConfig] = None,
        tool_call_id: Optional[str] = None,
        **kwargs: Any,
    ) -> Any:
        """Run the tool.

        Args:
            tool_input: The input to the tool.
            verbose: Whether to log the tool's progress. Defaults to None.
            start_color: The color to use when starting the tool. Defaults to 'green'.
            color: The color to use when ending the tool. Defaults to 'green'.
            callbacks: Callbacks to be called during tool execution. Defaults to None.
            tags: Optional list of tags associated with the tool. Defaults to None.
            metadata: Optional metadata associated with the tool. Defaults to None.
            run_name: The name of the run. Defaults to None.
            run_id: The id of the run. Defaults to None.
            config: The configuration for the tool. Defaults to None.
            tool_call_id: The id of the tool call. Defaults to None.
            kwargs: Keyword arguments to be passed to tool callbacks (event handler)

        Returns:
            The output of the tool.

        Raises:
            ToolException: If an error occurs during tool execution.
        """
        callback_manager = CallbackManager.configure(
            callbacks,
            self.callbacks,
            self.verbose or bool(verbose),
            tags,
            self.tags,
            metadata,
            self.metadata,
        )

        run_manager = callback_manager.on_tool_start(
            {"name": self.name, "description": self.description},
            tool_input if isinstance(tool_input, str) else str(tool_input),
            color=start_color,
            name=run_name,
            run_id=run_id,
            # Inputs by definition should always be dicts.
            # For now, it's unclear whether this assumption is ever violated,
            # but if it is we will send a `None` value to the callback instead
            # TODO: will need to address issue via a patch.
            inputs=tool_input if isinstance(tool_input, dict) else None,
            **kwargs,
        )

        content = None
        artifact = None
        status = "success"
        error_to_raise: Union[Exception, KeyboardInterrupt, None] = None
        try:
            child_config = patch_config(config, callbacks=run_manager.get_child())
            with set_config_context(child_config) as context:
                tool_args, tool_kwargs = self._to_args_and_kwargs(
                    tool_input, tool_call_id
                )
                if signature(self._run).parameters.get("run_manager"):
                    tool_kwargs |= {"run_manager": run_manager}
                if config_param := _get_runnable_config_param(self._run):
                    tool_kwargs |= {config_param: config}
                response = context.run(self._run, *tool_args, **tool_kwargs)
            if self.response_format == "content_and_artifact":
                if not isinstance(response, tuple) or len(response) != 2:
                    msg = (
                        "Since response_format='content_and_artifact' "
                        "a two-tuple of the message content and raw tool output is "
                        f"expected. Instead generated response of type: "
                        f"{type(response)}."
                    )
                    error_to_raise = ValueError(msg)
                else:
                    content, artifact = response
            else:
                content = response
        except (ValidationError, ValidationErrorV1) as e:
            if not self.handle_validation_error:
                error_to_raise = e
            else:
                content = _handle_validation_error(e, flag=self.handle_validation_error)
                status = "error"
        except ToolException as e:
            if not self.handle_tool_error:
                error_to_raise = e
            else:
                content = _handle_tool_error(e, flag=self.handle_tool_error)
                status = "error"
        except (Exception, KeyboardInterrupt) as e:
            error_to_raise = e

        if error_to_raise:
            run_manager.on_tool_error(error_to_raise)
            raise error_to_raise
        output = _format_output(content, artifact, tool_call_id, self.name, status)
        run_manager.on_tool_end(output, color=color, name=self.name, **kwargs)
        return output
```
####

run 代码有下面几个比较难以理解的地方：
1. `_to_args_and_kwargs`
2. `_run` 方法的参数兼容
    - `signature(self._run).parameters.get("run_manager")`
    - `config_param := _get_runnable_config_param(self._run)`
3. `_format_output`

#### _to_args_and_kwargs
`_to_args_and_kwargs`:
1. `_to_args_and_kwargs` 会调用 `_parse_input` 
2. `_parse_input` 会根据 `args_schema` 校验传入的 tool_input 参数
3. 其中会特殊处理 `InjectedToolCallId` 声明的字段，如果 args_schema 声明了这个类型的字段，但是没有传入这个参数，会把 tool_call_id 作为值传递进去
3. `_parse_input` 逻辑比较复杂，会判断 tool_input 的类型、args_schema 的类型，根据不同类型调用不同的校验方法

```python
tool_args, tool_kwargs = self._to_args_and_kwargs(
                    tool_input, tool_call_id
                )

    def _to_args_and_kwargs(
        self, tool_input: Union[str, dict], tool_call_id: Optional[str]
    ) -> tuple[tuple, dict]:
        """Convert tool input to positional and keyword arguments.

        Args:
            tool_input: The input to the tool.
            tool_call_id: The ID of the tool call, if available.

        Returns:
            A tuple of (positional_args, keyword_args) for the tool.

        Raises:
            TypeError: If the tool input type is invalid.
        """
        if (
            self.args_schema is not None
            and isinstance(self.args_schema, type)
            and is_basemodel_subclass(self.args_schema)
            and not get_fields(self.args_schema)
        ):
            # StructuredTool with no args
            return (), {}
        tool_input = self._parse_input(tool_input, tool_call_id)
        # For backwards compatibility, if run_input is a string,
        # pass as a positional argument.
        if isinstance(tool_input, str):
            return (tool_input,), {}
        if isinstance(tool_input, dict):
            # Make a shallow copy of the input to allow downstream code
            # to modify the root level of the input without affecting the
            # original input.
            # This is used by the tool to inject run time information like
            # the callback manager.
            return (), tool_input.copy()
        # This code path is not expected to be reachable.
        msg = f"Invalid tool input type: {type(tool_input)}"
        raise TypeError(msg)

    def _parse_input(
        self, tool_input: Union[str, dict], tool_call_id: Optional[str]
    ) -> Union[str, dict[str, Any]]:
        """Parse and validate tool input using the args schema.

        Args:
            tool_input: The raw input to the tool.
            tool_call_id: The ID of the tool call, if available.

        Returns:
            The parsed and validated input.

        Raises:
            ValueError: If string input is provided with JSON schema or if
                InjectedToolCallId is required but not provided.
            NotImplementedError: If args_schema is not a supported type.
        """
        input_args = self.args_schema
        if isinstance(tool_input, str):
            if input_args is not None:
                if isinstance(input_args, dict):
                    msg = (
                        "String tool inputs are not allowed when "
                        "using tools with JSON schema args_schema."
                    )
                    raise ValueError(msg)
                key_ = next(iter(get_fields(input_args).keys()))
                if issubclass(input_args, BaseModel):
                    input_args.model_validate({key_: tool_input})
                elif issubclass(input_args, BaseModelV1):
                    input_args.parse_obj({key_: tool_input})
                else:
                    msg = f"args_schema must be a Pydantic BaseModel, got {input_args}"
                    raise TypeError(msg)
            return tool_input
        if input_args is not None:
            if isinstance(input_args, dict):
                return tool_input
            if issubclass(input_args, BaseModel):
                for k, v in get_all_basemodel_annotations(input_args).items():
                    if (
                        _is_injected_arg_type(v, injected_type=InjectedToolCallId)
                        and k not in tool_input
                    ):
                        if tool_call_id is None:
                            msg = (
                                "When tool includes an InjectedToolCallId "
                                "argument, tool must always be invoked with a full "
                                "model ToolCall of the form: {'args': {...}, "
                                "'name': '...', 'type': 'tool_call', "
                                "'tool_call_id': '...'}"
                            )
                            raise ValueError(msg)
                        tool_input[k] = tool_call_id
                result = input_args.model_validate(tool_input)
                result_dict = result.model_dump()
            elif issubclass(input_args, BaseModelV1):
                for k, v in get_all_basemodel_annotations(input_args).items():
                    if (
                        _is_injected_arg_type(v, injected_type=InjectedToolCallId)
                        and k not in tool_input
                    ):
                        if tool_call_id is None:
                            msg = (
                                "When tool includes an InjectedToolCallId "
                                "argument, tool must always be invoked with a full "
                                "model ToolCall of the form: {'args': {...}, "
                                "'name': '...', 'type': 'tool_call', "
                                "'tool_call_id': '...'}"
                            )
                            raise ValueError(msg)
                        tool_input[k] = tool_call_id
                result = input_args.parse_obj(tool_input)
                result_dict = result.dict()
            else:
                msg = (
                    f"args_schema must be a Pydantic BaseModel, got {self.args_schema}"
                )
                raise NotImplementedError(msg)
            return {
                k: getattr(result, k) for k, v in result_dict.items() if k in tool_input
            }
        return tool_input
```

#### _run 方法的参数兼容
下面是一个示例:

```python
class MyTool(BaseTool):
    name = "my_tool"
    description = "测试工具"

    def _run(self, query: str, run_manager: CallbackManagerForToolRun, config: RunnableConfig) -> str:
        run_manager.on_text(f"收到 query: {query}")
        return f"查询完成: {query}"

```

1. `signature(self._run).parameters.get("run_manager")`: 检查有没有 run_manager 命名的参数
`config_param := _get_runnable_config_param(self._run)`: 检查有没有 RunnableConfig 注解的参数

#### _format_output
_format_output 会将 _run 的结果包装为 ToolMessage

```python
def _format_output(
    content: Any,
    artifact: Any,
    tool_call_id: Optional[str],
    name: str,
    status: str,
) -> Union[ToolOutputMixin, Any]:
    """Format tool output as a ToolMessage if appropriate.

    Args:
        content: The main content of the tool output.
        artifact: Any artifact data from the tool.
        tool_call_id: The ID of the tool call.
        name: The name of the tool.
        status: The execution status.

    Returns:
        The formatted output, either as a ToolMessage or the original content.
    """
    if isinstance(content, ToolOutputMixin) or tool_call_id is None:
        return content
    if not _is_message_content_type(content):
        content = _stringify(content)
    return ToolMessage(
        content,
        artifact=artifact,
        tool_call_id=tool_call_id,
        name=name,
        status=status,
    )
```

## 3. SimpleTool
SimpleTool 是基于函数封装的工具。
1. `_to_args_and_kwargs` 函数限定 tool_input 只能有一个参数，多参数要使用 StructuredTool
2. tool_input 只能是单参数，但是传入的函数可以额外接收 callbacks 和 RunnableConfig

```python
class Tool(BaseTool):
    """Tool that takes in function or coroutine directly."""

    description: str = ""
    func: Optional[Callable[..., str]]
    """The function to run when the tool is called."""
    coroutine: Optional[Callable[..., Awaitable[str]]] = None
    """The asynchronous version of the function."""

    def _to_args_and_kwargs(
        self, tool_input: Union[str, dict], tool_call_id: Optional[str]
    ) -> tuple[tuple, dict]:
        """Convert tool input to pydantic model."""
        args, kwargs = super()._to_args_and_kwargs(tool_input, tool_call_id)
        # For backwards compatibility. The tool must be run with a single input
        all_args = list(args) + list(kwargs.values())
        if len(all_args) != 1:
            msg = (
                f"""Too many arguments to single-input tool {self.name}.
                Consider using StructuredTool instead."""
                f" Args: {all_args}"
            )
            raise ToolException(msg)
        return tuple(all_args), {}

    def _run(
        self,
        *args: Any,
        config: RunnableConfig,
        run_manager: Optional[CallbackManagerForToolRun] = None,
        **kwargs: Any,
    ) -> Any:
        """Use the tool."""
        if self.func:
            if run_manager and signature(self.func).parameters.get("callbacks"):
                kwargs["callbacks"] = run_manager.get_child()
            if config_param := _get_runnable_config_param(self.func):
                kwargs[config_param] = config
            return self.func(*args, **kwargs)
        msg = "Tool does not support sync invocation."
        raise NotImplementedError(msg)
```

## 4. StructuredTool 
StructuredTool 也是基于函数的 Tool 实现。复杂主要复杂在函数元数据的解析上，实现位于 from_function 方法内。


```python
class StructuredTool(BaseTool):
    """Tool that can operate on any number of inputs."""

    description: str = ""
    args_schema: Annotated[ArgsSchema, SkipValidation()] = Field(
        ..., description="The tool schema."
    )
    """The input arguments' schema."""
    func: Optional[Callable[..., Any]] = None
    """The function to run when the tool is called."""
    coroutine: Optional[Callable[..., Awaitable[Any]]] = None
    """The asynchronous version of the function."""

    @classmethod
    def from_function(
        cls,
        func: Optional[Callable] = None,
        coroutine: Optional[Callable[..., Awaitable[Any]]] = None,
        name: Optional[str] = None,
        description: Optional[str] = None,
        return_direct: bool = False,  # noqa: FBT001,FBT002
        args_schema: Optional[ArgsSchema] = None,
        infer_schema: bool = True,  # noqa: FBT001,FBT002
        *,
        response_format: Literal["content", "content_and_artifact"] = "content",
        parse_docstring: bool = False,
        error_on_invalid_docstring: bool = False,
        **kwargs: Any,
    ) -> StructuredTool:
```


from_function:

> **将一个普通函数（带类型注解和 docstring）转换为 StructuredTool 实例**，并自动处理参数 schema 推断、描述生成等逻辑。

换句话说，它实现了从：

```python
def add(a: int, b: int) -> int:
    """Add two numbers."""
    return a + b
```

变成：

```python
StructuredTool(
    name="add",
    func=add,
    args_schema=自动生成的 Pydantic schema,
    description="Add two numbers."
)
```

---

### ✅ 参数说明

| 参数名                          | 说明                                                   |
| ---------------------------- | ---------------------------------------------------- |
| `func`                       | 要转换为 Tool 的普通函数（sync）                                |
| `coroutine`                  | 异步函数（async），和 `func` 二选一                             |
| `name`                       | Tool 的名称，默认用函数名                                      |
| `description`                | Tool 的描述，默认用函数 docstring                             |
| `return_direct`              | 是否直接返回结果（跳过 Agent 的输出封装）                             |
| `args_schema`                | Tool 输入参数的 Pydantic Schema，可以手动提供                    |
| `infer_schema`               | 是否自动从函数签名推导 schema（默认开启）                             |
| `response_format`            | `content` 或 `content_and_artifact`，控制 ToolMessage 格式 |
| `parse_docstring`            | 是否解析 docstring 的参数说明（Google Style）                   |
| `error_on_invalid_docstring` | docstring 解析失败是否报错                                   |
| `kwargs`                     | 传给 StructuredTool 的额外参数                              |

---

### ✅ 核心逻辑拆解

源码的核心步骤可以分为 6 步：

---

1️⃣ 获取目标函数对象：

```python
if func is not None:
    source_function = func
elif coroutine is not None:
    source_function = coroutine
else:
    raise ValueError("Function and/or coroutine must be provided")
```

必须传入 `func` 或 `coroutine`，二选一。

---

2️⃣ 设置 name 和 args\_schema：

```python
name = name or source_function.__name__

if args_schema is None and infer_schema:
    args_schema = create_schema_from_function(...)
```

若未手动传入 `args_schema`，则自动调用 `create_schema_from_function` 将函数签名转成一个 `Pydantic.BaseModel` 类。

---

3️⃣ 获取 Tool 描述 description：

优先级：

1. 使用传入的 `description`
2. 若没传，则使用函数 docstring（`__doc__`）
3. 若没 docstring，尝试用 `args_schema.__doc__`
4. 最后都没提供时，报错

---

4️⃣ 整理描述格式：

```python
description_ = textwrap.dedent(description_).strip()
description_ = f"{description_.strip()}"
```

对 docstring 做了标准化处理，防止缩进问题。

---

5️⃣ 最终构造 Tool 对象

```python
return cls(
    name=name,
    func=func,
    coroutine=coroutine,
    args_schema=args_schema,
    description=description_,
    return_direct=return_direct,
    response_format=response_format,
    **kwargs,
)
```

调用 `StructuredTool.__init__` 构建实例返回。

---

✅ 使用示例

```python
from langchain.tools import StructuredTool

def add(a: int, b: int) -> int:
    """Add two numbers."""
    return a + b

tool = StructuredTool.from_function(add)
result = tool.invoke({"a": 2, "b": 3})
print(result)  # 输出：5
```

## 5. Convert
Tool 还有最后一部分内容 Convert.py，提供了各种工具，将 Function、Runnable 转换为 Tool。

Convert 内主要提供的是 tool 函数，这个函数内部实现的是一个带可选参数的装饰器。内部会完成以下逻辑:
1. 判断是 Function 还是 Runnable
2. 根据参数个数等选择 StructuredTool 还是 Tool
3. 调用 from_function 方法构建 Tool 实例

具体代码不贴了，举一些使用示例:

```python
from langchain_core.tools import tool

@tool
def add(a: int, b: int) -> int:
    """Add two integers."""
    return a + b

# 调用方式
result = add.invoke({"a": 1, "b": 2})
print(result)  # 输出: 3


@tool(parse_docstring=True)
def greet(name: str) -> str:
    """Greet someone by name.

    Args:
        name: The name of the person to greet.
    """
    return f"Hello, {name}!"

# 转换一个 Runnable
runnable = RunnableLambda(lambda x: f"Echo: {x}")

echo_tool = tool("echo", runnable)
print(echo_tool.invoke("hello"))  # 输出: Echo: hello

# 自动推断 Runnable 输入类型
from typing import TypedDict
from langchain_core.runnables import RunnableLambda

class Input(TypedDict):
    name: str
    age: int

def greet_user(data: Input) -> str:
    return f"Hi {data['name']}, age {data['age']}"

tool_greet = tool("greet", RunnableLambda(greet_user))
print(tool_greet.invoke({"name": "Alice", "age": 30}))
```

## 6. 总结
在 Tool 我们介绍了如下内容:
1. AIMessage: 包含了模型返回的 Tool Call 信息
2. Function 和 Tool 相关的 OutPutParser 用于从 AIMessage 解析出 Call 相关参数
3. 我们介绍了如何构造 Tool

我们看一个组合所有内容的示例:

```python
from audioop import mul
from langchain_core.output_parsers.openai_tools import PydanticToolsParser
from langchain_core.messages import AIMessage, ToolCall
from langchain_core.outputs import ChatGeneration
from langchain_core.tools import tool
from pydantic import BaseModel


# 1. 定义工具输入模型
class MultiplyInput(BaseModel):
    x: int
    y: int


class AddInput(BaseModel):
    a: int
    b: int


# 2. 使用 @tool 装饰函数
@tool
def multiply_tool(input: MultiplyInput) -> str:
    """计算乘法"""
    return str(input.x * input.y)


@tool
def add_tool(input: AddInput) -> str:
    """计算加法"""
    return str(input.a + input.b)


# ✅ 3. 从 StructuredTool 拿到绑定后的 Pydantic 模型类
multiply_model = multiply_tool.args_schema
add_model = add_tool.args_schema
print(multiply_model.__name__, multiply_model.model_json_schema())

# 4. 模拟 LLM 返回 tool 调用结果
fake_tool_call = ToolCall(
    # 注意这里的 args 
    name="multiply_tool", args={"input": {"x": 6, "y": 7}}, id="tool_call_1"
)
ai_message = AIMessage(content="", tool_calls=[fake_tool_call])
generation = ChatGeneration(message=ai_message)

# ✅ 5. 使用 PydanticToolsParser（传入的是 Pydantic 模型类）
parser = PydanticToolsParser(tools=[multiply_model, add_model])
parsed = parser.parse_result([generation])

# 6. 输出结构化结果
print("✅ 解析后的 Pydantic 对象:")
print(parsed)
```
