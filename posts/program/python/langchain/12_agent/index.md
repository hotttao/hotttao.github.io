# LangChain Agent


有了 langchain-core 基础后，现在可以学习 Agent 的实现了。

## 1. Agent UML 类图
Agent 的实现在 langchain.agents 中，UML 类图如下:

![Agent](/images/langchain/agent.svg)

UML 类图中，我们能看到很多之前在 langchain-core 学习的类，我们需要重点关注一下对象:

```bash
ChatPromptTemplate
    AgentScratchPadChatPromptTemplate

BaseOutputParser
    MultiActionAgentOutputParser
    AgentOutputParser
        ChatOutputParser

BaseMultiActionAgent
    RunnableMultiActionAgent
BaseSingleActionAgent
    RunnableAgent
    LLMSingleActionAgent
    Agent
        ChatAgent

Chain
  AgentExecutor
```

我们来直接问问大模型，这些类的作用。为了避免混淆这次我们分开来提问。

### 1.1 ActionAgent
```python
我正在阅读  langchain.agents 子包的源代码，注意到包内，有如下类:
BaseMultiActionAgent
    RunnableMultiActionAgent
BaseSingleActionAgent
    RunnableAgent
    LLMSingleActionAgent
    Agent
        ChatAgent

请给我从语义上解释一下这些类的的作用
```

下面是回答:

你提到的这些类属于 `langchain.agents` 模块，主要用于定义各种 **Agent 类型** 的抽象结构与具体实现。LangChain 中的 Agent 是一种根据输入动态决定下一步操作（例如调用哪个工具）的逻辑单元。在 LCEL 架构中，它们也实现为 `Runnable`，可以像其他组件一样组合和运行。

我将按照从抽象到具体的层次，逐个解释这些类的语义与作用，并结合它们在 LCEL 中的地位说明其职责。


LangChain Agent 按照 **是否支持一次调用多个工具**，可以分为两类：

| 分类                      | 描述             |
| ----------------------- | -------------- |
| **Single Action Agent** | 每次决策只执行一个工具调用  |
| **Multi Action Agent**  | 每次决策可以返回多个工具调用 |

#### 1. `BaseSingleActionAgent` (抽象类)

* **作用**：定义只执行一个工具调用的 Agent 的基础行为。
* **职责**：

  * 实现 `plan()` 方法：根据输入和上下文信息决定接下来要调用的工具。
  * 只支持单次调用（如 Tool A）后等待结果，不支持并发或多调用。

---

#### 2. `BaseMultiActionAgent` (抽象类)

* **作用**：支持每轮执行多个工具调用的 Agent 的基础类。
* **职责**：

  * 实现 `plan()`，但返回的是多个 ToolCall。
  * 适合需要并行调用多个工具的情况，例如：`Tool A`, `Tool B` 同时使用，随后统一处理结果。

---

#### 3. `RunnableAgent` (具体实现)

* **继承**：`BaseSingleActionAgent`
* **作用**：将旧式 Agent 包装成符合 LCEL `Runnable` 接口的对象。
* **职责**：

  * 实现 LCEL 所需的 `invoke`/`stream` 方法，使其可以与其他 Runnable 组件组合使用。
  * 通常接受一个 `AgentExecutor` 或类似的 agent 逻辑实例。

---

#### 4. `RunnableMultiActionAgent` (具体实现)

* **继承**：`BaseMultiActionAgent`
* **作用**：将多工具调用的 Agent 实现为 LCEL 中可运行的 `Runnable`。
* **职责**：

  * 可并行触发多个工具的调用，适合复杂决策场景。

---

#### 5. `Agent` (抽象类)

* **继承**：`BaseSingleActionAgent`
* **作用**：早期 agent 系统的通用基础类，现多由 `RunnableAgent` 替代使用。
* **职责**：

  * 实现单次推理和工具调用逻辑。
  * 支持 Agent 的 scratchpad 概念（历史调用记录）。

---

#### 6. `LLMSingleActionAgent` (具体实现)

* **继承**：`Agent`
* **作用**：用 LLM 来预测下一步应该调用哪个工具的 Agent。
* **职责**：

  * 解析 prompt 输出，构造工具调用（ToolCall）。
  * 常用于基于 OpenAI ChatCompletion 格式的 Tool Calling。

---

#### 7. `ChatAgent` (具体实现)

* **继承**：`Agent`
* **作用**：使用 Chat 模型（如 gpt-4）的 Agent 实现。
* **职责**：

  * 基于 ChatMessages（而非纯文本）进行推理。
  * 通常配合 `OpenAICallbackHandler` 和工具列表。

---



| 类名                         | 使用场景示例                           |
| -------------------------- | -------------------------------- |
| `LLMSingleActionAgent`     | 基于 LLM 的 Tool Calling（旧版，非 LCEL） |
| `ChatAgent`                | 基于 Chat 模型的 Tool Calling         |
| `RunnableAgent`            | LCEL 中将现有 Agent 接入链路             |
| `RunnableMultiActionAgent` | 支持多工具并发调用的 Agent（如多模态任务）         |


### 1.2 AgentAction
langchain_core.agents 提供了 `AgentAction` 类:

```bash
AgentAction
    AgentActionMessageLog
AgentStep
AgentFinish
```

```bash
我正在阅读  langchain_core.agents 子包的源代码，注意到包内，有如下类:
AgentAction
    AgentActionMessageLog
AgentStep
AgentFinish

请给我从语义上解释一下这些类的的作用
```

下面是回答:

你提到的这些类属于 `langchain_core.agents` 子包，它们是 LangChain 中 **Agent 推理过程的数据结构模型**，用于表示 Agent 在每一步的决策、工具调用和最终结果的结构化信息。

这些类大多继承自 `BaseModel`（Pydantic），不属于 Agent 本身的执行逻辑，而是用于表示 Agent 决策的 **中间状态与最终结果**，在 tracing、执行记录、回调、日志记录中非常关键。



| 类名                      | 类型  | 作用简述                                 |
| ----------------------- | --- | ------------------------------------ |
| `AgentAction`           | 基类  | 表示 Agent 要调用的工具及其参数                  |
| `AgentActionMessageLog` | 子类  | 在 `AgentAction` 基础上记录调用时的 message 日志 |
| `AgentStep`             | 包装类 | 包装了一次 AgentAction 和其对应的工具调用结果        |
| `AgentFinish`           | 终止类 | 表示 Agent 推理完成的最终输出                   |

---


#### 1. `AgentAction`

```python
class AgentAction(BaseModel):
    tool: str
    tool_input: Any
    log: str = ""
```

✅ 作用：

* 表示 Agent 做出的一次 **工具调用决策**，即：“我要调用哪个工具，用什么参数？”
* 是 Agent 推理过程中每一步产生的结果之一。

✅ 字段说明：

| 字段名          | 类型    | 说明                            |
| ------------ | ----- | ----------------------------- |
| `tool`       | `str` | 工具名称（字符串标识）                   |
| `tool_input` | `Any` | 工具输入的参数，通常是一个 dict            |
| `log`        | `str` | 可选的文本日志，描述这次决策的 rationale（理由） |

---

#### 2. `AgentActionMessageLog`（`AgentAction` 的扩展）

```python
class AgentActionMessageLog(AgentAction):
    message_log: list[BaseMessage]
```

✅ 作用：

* 是 `AgentAction` 的子类，在调用信息之外，还额外保留了用于推理的 **消息上下文（message\_log）**。
* 适用于基于 ChatModel 的 Agent，如 ChatGPT 风格的调用。

✅ 字段新增：

| 字段名           | 类型                  | 说明                             |
| ------------- | ------------------- | ------------------------------ |
| `message_log` | `list[BaseMessage]` | Agent 推理时的对话上下文，用于回溯判断为何调用某个工具 |

---

#### 3. `AgentStep`

```python
class AgentStep(BaseModel):
    action: AgentAction
    observation: str
```

✅ 作用：

* 表示 Agent 执行了一次工具调用（`action`），然后得到了结果（`observation`）。
* 是 Agent 决策轨迹（Scratchpad）中的核心数据结构之一。

✅ 字段说明：

| 字段名           | 类型            | 说明                   |
| ------------- | ------------- | -------------------- |
| `action`      | `AgentAction` | 当前这步 agent 要做什么      |
| `observation` | `str`         | 工具返回的结果或观察信息（如函数返回值） |

---

#### 4. `AgentFinish`

```python
class AgentFinish(BaseModel):
    return_values: dict[str, Any]
    log: str = ""
```

✅ 作用：

* 表示 Agent 已完成所有工具调用，进入 **最终输出阶段**。
* 在 LCEL 或经典 AgentExecutor 中，表示整个链路流程结束。

✅ 字段说明：

| 字段名             | 类型               | 说明                             |
| --------------- | ---------------- | ------------------------------ |
| `return_values` | `dict[str, Any]` | 最终返回的键值对，通常是 `{"output": ...}` |
| `log`           | `str`            | 日志说明，可以用于 UI 展示或 trace 分析      |

---

#### 🧠 总结对比表

| 类名                      | 表示含义                        | 常见用途                        |
| ----------------------- | --------------------------- | --------------------------- |
| `AgentAction`           | 一次工具调用的决策                   | 推理阶段输出                      |
| `AgentActionMessageLog` | 附带上下文的工具调用决策                | ChatModel Agent             |
| `AgentStep`             | 一个 action + observation 的组合 | Scratchpad、run history      |
| `AgentFinish`           | Agent 推理完成后的最终输出            | `AgentExecutor` 或 LCEL 的返回值 |

### 1.3 AgentExecutor
提问: AgentExecutor 是什么
回答:

`AgentExecutor` 是 LangChain 中用来 **驱动 Agent 推理循环并执行工具调用的核心组件**，它负责：

> **根据 Agent 的决策（`AgentAction`）不断调用工具，直到 Agent 给出最终输出（`AgentFinish`）为止。**

可以看作是 **Agent 的控制器或运行时引擎**，配合 Agent 本体（如 `RunnableAgent`）和工具（`Tool`)）共同组成完整的智能体系统。

🧠 一句话定义

> `AgentExecutor` = Agent（做决策） + 工具列表（执行动作） + 执行控制循环


🧩 类图关系简化

```text
User Input
   ↓
AgentExecutor
   ↓         ↘
Agent      Tools
   ↓         ↓
Action → ToolCall → Observation
   ↑                         ↓
Finish (output) ← Observation
```


## 2. BaseSingleActionAgent
BaseSingleActionAgent 核心抽象方法是 plan，在输入的工具中，决定使用哪一个。

```python
class BaseSingleActionAgent(BaseModel):
    """Base Single Action Agent class."""

    @property
    def return_values(self) -> list[str]:
        """Return values of the agent."""
        return ["output"]

    @abstractmethod
    def plan(
        self,
        intermediate_steps: list[tuple[AgentAction, str]],
        callbacks: Callbacks = None,
        **kwargs: Any,
    ) -> Union[AgentAction, AgentFinish]:
        """Given input, decided what to do.

        Args:
            intermediate_steps: Steps the LLM has taken to date,
                along with observations.
            callbacks: Callbacks to run.
            **kwargs: User inputs.

        Returns:
            Action specifying what tool to use.
        """

    @abstractmethod
    async def aplan(
        self,
        intermediate_steps: list[tuple[AgentAction, str]],
        callbacks: Callbacks = None,
        **kwargs: Any,
    ) -> Union[AgentAction, AgentFinish]:
        """Async given input, decided what to do.

        Args:
            intermediate_steps: Steps the LLM has taken to date,
                along with observations.
            callbacks: Callbacks to run.
            **kwargs: User inputs.

        Returns:
            Action specifying what tool to use.
        """

    @property
    @abstractmethod
    def input_keys(self) -> list[str]:
        """Return the input keys.

        :meta private:
        """
```

### 2.1 RunnableAgent

```python
class RunnableAgent(BaseSingleActionAgent):
    """Agent powered by Runnables."""

    runnable: Runnable[dict, Union[AgentAction, AgentFinish]]
    """Runnable to call to get agent action."""
    input_keys_arg: list[str] = []
    return_keys_arg: list[str] = []
    stream_runnable: bool = True

    def plan(
        self,
        intermediate_steps: list[tuple[AgentAction, str]],
        callbacks: Callbacks = None,
        **kwargs: Any,
    ) -> Union[AgentAction, AgentFinish]:
        """Based on past history and current inputs, decide what to do.

        Args:
            intermediate_steps: Steps the LLM has taken to date,
                along with the observations.
            callbacks: Callbacks to run.
            **kwargs: User inputs.

        Returns:
            Action specifying what tool to use.
        """
        inputs = {**kwargs, **{"intermediate_steps": intermediate_steps}}
        final_output: Any = None
        if self.stream_runnable:
            # Use streaming to make sure that the underlying LLM is invoked in a
            # streaming
            # fashion to make it possible to get access to the individual LLM tokens
            # when using stream_log with the Agent Executor.
            # Because the response from the plan is not a generator, we need to
            # accumulate the output into final output and return that.
            for chunk in self.runnable.stream(inputs, config={"callbacks": callbacks}):
                if final_output is None:
                    final_output = chunk
                else:
                    final_output += chunk
        else:
            final_output = self.runnable.invoke(inputs, config={"callbacks": callbacks})

        return final_output
```

### 2.2 LLMSingleActionAgent
```python
class LLMSingleActionAgent(BaseSingleActionAgent):
    """Base class for single action agents."""

    llm_chain: LLMChain
    """LLMChain to use for agent."""
    output_parser: AgentOutputParser
    """Output parser to use for agent."""
    stop: list[str]
    """List of strings to stop on."""

    def plan(
        self,
        intermediate_steps: list[tuple[AgentAction, str]],
        callbacks: Callbacks = None,
        **kwargs: Any,
    ) -> Union[AgentAction, AgentFinish]:
        """Given input, decided what to do.

        Args:
            intermediate_steps: Steps the LLM has taken to date,
                along with the observations.
            callbacks: Callbacks to run.
            **kwargs: User inputs.

        Returns:
            Action specifying what tool to use.
        """
        output = self.llm_chain.run(
            intermediate_steps=intermediate_steps,
            stop=self.stop,
            callbacks=callbacks,
            **kwargs,
        )
        return self.output_parser.parse(output)
```

### 2.3 Agent
Agent 提供了默认的 plan 实现单次推理和工具调用逻辑。他定义的抽象方法主要是为了获取 prompt 模板和 output_parser。

```python
@deprecated(
    "0.1.0",
    message=AGENT_DEPRECATION_WARNING,
    removal="1.0",
)
class Agent(BaseSingleActionAgent):
    """Agent that calls the language model and deciding the action.

    This is driven by a LLMChain. The prompt in the LLMChain MUST include
    a variable called "agent_scratchpad" where the agent can put its
    intermediary work.
    """

    llm_chain: LLMChain
    """LLMChain to use for agent."""
    output_parser: AgentOutputParser
    """Output parser to use for agent."""
    allowed_tools: Optional[list[str]] = None
    """Allowed tools for the agent. If None, all tools are allowed."""

    def plan(
        self,
        intermediate_steps: list[tuple[AgentAction, str]],
        callbacks: Callbacks = None,
        **kwargs: Any,
    ) -> Union[AgentAction, AgentFinish]:
        """Given input, decided what to do.

        Args:
            intermediate_steps: Steps the LLM has taken to date,
                along with observations.
            callbacks: Callbacks to run.
            **kwargs: User inputs.

        Returns:
            Action specifying what tool to use.
        """
        full_inputs = self.get_full_inputs(intermediate_steps, **kwargs)
        full_output = self.llm_chain.predict(callbacks=callbacks, **full_inputs)
        return self.output_parser.parse(full_output)

    @property
    @abstractmethod
    def observation_prefix(self) -> str:
        """Prefix to append the observation with."""

    @property
    @abstractmethod
    def llm_prefix(self) -> str:
        """Prefix to append the LLM call with."""

    @classmethod
    @abstractmethod
    def create_prompt(cls, tools: Sequence[BaseTool]) -> BasePromptTemplate:
        pass
    
    @classmethod
    @abstractmethod
    def _get_default_output_parser(cls, **kwargs: Any) -> AgentOutputParser:
        """Get default output parser for this class."""
        pass
```

## 3. BaseMultiActionAgent

```python
class BaseMultiActionAgent(BaseModel):
    """Base Multi Action Agent class."""

    @abstractmethod
    def plan(
        self,
        intermediate_steps: list[tuple[AgentAction, str]],
        callbacks: Callbacks = None,
        **kwargs: Any,
    ) -> Union[list[AgentAction], AgentFinish]:
        """Given input, decided what to do.

        Args:
            intermediate_steps: Steps the LLM has taken to date,
                along with the observations.
            callbacks: Callbacks to run.
            **kwargs: User inputs.

        Returns:
            Actions specifying what tool to use.
        """


    @property
    @abstractmethod
    def input_keys(self) -> list[str]:
        """Return the input keys.

        :meta private:
        """
```

### 3.1 RunnableMultiActionAgent
1. RunnableMultiActionAgent 输入是 `Runnable[dict, Union[list[AgentAction], AgentFinish]]`
2. RunnableAgent 输入是 `Runnable[dict, Union[AgentAction, AgentFinish]]`
3. 两个类默认提供的 plan 方法完全相同。

```python 
class RunnableMultiActionAgent(BaseMultiActionAgent):
    """Agent powered by Runnables."""

    runnable: Runnable[dict, Union[list[AgentAction], AgentFinish]]
    """Runnable to call to get agent actions."""
    input_keys_arg: list[str] = []
    return_keys_arg: list[str] = []
    stream_runnable: bool = True
    """Whether to stream from the runnable or not.
```

## 4. AgentExecutor

### 4.1 属性

```python
class AgentExecutor(Chain):
    """Agent that is using tools."""

    agent: Union[BaseSingleActionAgent, BaseMultiActionAgent, Runnable]
    """The agent to run for creating a plan and determining actions
    to take at each step of the execution loop."""
    tools: Sequence[BaseTool]
    """The valid tools the agent can call."""
    return_intermediate_steps: bool = False
    """Whether to return the agent's trajectory of intermediate steps
    at the end in addition to the final output."""
    max_iterations: Optional[int] = 15
    """The maximum number of steps to take before ending the execution
    loop.

    Setting to 'None' could lead to an infinite loop."""
    max_execution_time: Optional[float] = None
    """The maximum amount of wall clock time to spend in the execution
    loop.
    """
    early_stopping_method: str = "force"
    """The method to use for early stopping if the agent never
    returns `AgentFinish`. Either 'force' or 'generate'.

    `"force"` returns a string saying that it stopped because it met a
        time or iteration limit.

    `"generate"` calls the agent's LLM Chain one final time to generate
        a final answer based on the previous steps.
    """
    handle_parsing_errors: Union[bool, str, Callable[[OutputParserException], str]] = (
        False
    )
    """How to handle errors raised by the agent's output parser.
    Defaults to `False`, which raises the error.
    If `true`, the error will be sent back to the LLM as an observation.
    If a string, the string itself will be sent to the LLM as an observation.
    If a callable function, the function will be called with the exception
     as an argument, and the result of that function will be passed to the agent
      as an observation.
    """
    trim_intermediate_steps: Union[
        int, Callable[[list[tuple[AgentAction, str]]], list[tuple[AgentAction, str]]]
    ] = -1
    """How to trim the intermediate steps before returning them.
    Defaults to -1, which means no trimming.
    """
```

以下是 `AgentExecutor` 类中各个属性的语义解释表格：

| 属性名                         | 类型                                                             | 说明                                                                                                                            |
| --------------------------- | -------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| `agent`                     | `Union[BaseSingleActionAgent, BaseMultiActionAgent, Runnable]` | 核心 Agent 对象，负责根据当前状态和输入决定下一步操作（调用哪个 Tool 或结束）                                                                                 |
| `tools`                     | `Sequence[BaseTool]`                                           | Agent 可用的工具列表，每个工具通常是一个函数或 API 调用的封装                                                                                          |
| `return_intermediate_steps` | `bool`                                                         | 是否返回中间推理步骤（即每次调用 tool 的 action + observation 轨迹）                                                                              |
| `max_iterations`            | `Optional[int]`                                                | 最大允许的 Agent 执行步数，防止死循环（默认最多 15 步）                                                                                             |
| `max_execution_time`        | `Optional[float]`                                              | 最大运行时间（单位为秒），超过此时间则提前终止                                                                                                       |
| `early_stopping_method`     | `str`（`"force"` 或 `"generate"`）                                | Agent 在未主动结束时的终止策略：<br>`force` 直接终止并返回提示，<br>`generate` 让 agent 再调用一次 LLM 总结出结果                                               |
| `handle_parsing_errors`     | `Union[bool, str, Callable]`                                   | 控制当输出解析器（output parser）出错时的处理方式：<br>`False` 抛出异常；<br>`True` 把错误反馈给 LLM；<br>`str` 作为 observation 给 LLM；<br>`Callable` 动态生成反馈内容 |
| `trim_intermediate_steps`   | `int` 或 `Callable`                                             | 控制返回的中间步骤数量：<br>`-1` 不裁剪；<br>`int` 倒数保留 N 个；<br>`Callable` 自定义裁剪逻辑                                                            |


### 4.2 _call 方法

因为 AgentExecutor 继承子 Chain 所以其核心实现位于 _call 方法中。

```bash
用户输入 → _call()
  ↓
构建工具映射表
  ↓
初始化步骤计数器和时间
  ↓
WHILE 未超时/超迭代:
    → Agent 决策 (AgentAction)
    → 工具调用 (tool_input → output)
    → 添加中间步骤
    → 检查是否提前返回
  ↓
超限退出时用 early_stopping_method 处理
  ↓
格式化返回（包含中间轨迹）

```

Agent 决策，工具调用位于 _take_next_step 函数

```python
    def _call(
        self,
        inputs: dict[str, str],
        run_manager: Optional[CallbackManagerForChainRun] = None,
    ) -> dict[str, Any]:
        """Run text through and get agent response."""
        # Construct a mapping of tool name to tool for easy lookup
        name_to_tool_map = {tool.name: tool for tool in self.tools}
        # We construct a mapping from each tool to a color, used for logging.
        color_mapping = get_color_mapping(
            [tool.name for tool in self.tools], excluded_colors=["green", "red"]
        )
        intermediate_steps: list[tuple[AgentAction, str]] = []
        # Let's start tracking the number of iterations and time elapsed
        iterations = 0
        time_elapsed = 0.0
        start_time = time.time()
        # We now enter the agent loop (until it returns something).
        while self._should_continue(iterations, time_elapsed):
            next_step_output = self._take_next_step(
                name_to_tool_map,
                color_mapping,
                inputs,
                intermediate_steps,
                run_manager=run_manager,
            )
            if isinstance(next_step_output, AgentFinish):
                return self._return(
                    next_step_output, intermediate_steps, run_manager=run_manager
                )

            intermediate_steps.extend(next_step_output)
            if len(next_step_output) == 1:
                next_step_action = next_step_output[0]
                # See if tool should return directly
                tool_return = self._get_tool_return(next_step_action)
                if tool_return is not None:
                    return self._return(
                        tool_return, intermediate_steps, run_manager=run_manager
                    )
            iterations += 1
            time_elapsed = time.time() - start_time
        output = self._action_agent.return_stopped_response(
            self.early_stopping_method, intermediate_steps, **inputs
        )
        return self._return(output, intermediate_steps, run_manager=run_manager)

```

### 4.3 _take_next_step
_perform_agent_action 是实际调用 Tool

```python
    def _iter_next_step(
        self,
        name_to_tool_map: dict[str, BaseTool],
        color_mapping: dict[str, str],
        inputs: dict[str, str],
        intermediate_steps: list[tuple[AgentAction, str]],
        run_manager: Optional[CallbackManagerForChainRun] = None,
    ) -> Iterator[Union[AgentFinish, AgentAction, AgentStep]]:
        """Take a single step in the thought-action-observation loop.

        Override this to take control of how the agent makes and acts on choices.
        """
        try:
            intermediate_steps = self._prepare_intermediate_steps(intermediate_steps)

            # Call the LLM to see what to do.
            output = self._action_agent.plan(
                intermediate_steps,
                callbacks=run_manager.get_child() if run_manager else None,
                **inputs,
            )
        except OutputParserException as e:
            # 定义错误如何处理
            ...
            yield AgentStep(action=output, observation=observation)
            return

        # If the tool chosen is the finishing tool, then we end and return.
        if isinstance(output, AgentFinish):
            yield output
            return

        actions: list[AgentAction]
        if isinstance(output, AgentAction):
            actions = [output]
        else:
            actions = output
        # 返回 LLM 选择的 tool
        for agent_action in actions:
            yield agent_action

        #  返回这些 tool 的调用结果
        for agent_action in actions:
            yield self._perform_agent_action(
                name_to_tool_map, color_mapping, agent_action, run_manager
            )
```


### 4.4 _perform_agent_action
调用 tool.run 获取 tool 运行结果。

```python
    def _perform_agent_action(
        self,
        name_to_tool_map: dict[str, BaseTool],
        color_mapping: dict[str, str],
        agent_action: AgentAction,
        run_manager: Optional[CallbackManagerForChainRun] = None,
    ) -> AgentStep:
        if run_manager:
            run_manager.on_agent_action(agent_action, color="green")
        # Otherwise we lookup the tool
        if agent_action.tool in name_to_tool_map:
            tool = name_to_tool_map[agent_action.tool]
            return_direct = tool.return_direct
            color = color_mapping[agent_action.tool]
            tool_run_kwargs = self._action_agent.tool_run_logging_kwargs()
            if return_direct:
                tool_run_kwargs["llm_prefix"] = ""
            # We then call the tool on the tool input to get an observation
            observation = tool.run(
                agent_action.tool_input,
                verbose=self.verbose,
                color=color,
                callbacks=run_manager.get_child() if run_manager else None,
                **tool_run_kwargs,
            )
        else:
            tool_run_kwargs = self._action_agent.tool_run_logging_kwargs()
            observation = InvalidTool().run(
                {
                    "requested_tool_name": agent_action.tool,
                    "available_tool_names": list(name_to_tool_map.keys()),
                },
                verbose=self.verbose,
                color=None,
                callbacks=run_manager.get_child() if run_manager else None,
                **tool_run_kwargs,
            )
        return AgentStep(action=agent_action, observation=observation)
```

### 4.5 调用链
总结一下，AgentExecutor 的调用链如下:

```bash
invoke
    _call
        _take_next_step
            action_agent.plan
            _perform_agent_action
                tool.run
```
