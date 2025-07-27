# LangChain Message



## 1. Message 的 UML 类图
![messages 类图](/images/langchain/messages-all.svg)

从 UML 类图可以看出，Message 类是一个抽象类，它有三个子类：
1. BaseMessage
2. BaseMessageChunk
3. BaseDataContentBlock

### 1.1 提问
让我们问问大模型这三个基类代表的语义。

```bash
我正在阅读  langchain message 子包的源代码，注意到 message 包内，有三个 base 的抽象类:
1. BaseMessage
2. BaseMessageChunk
3. BaseDataContentBlock
请给我从语义上解释一下这三个类的的作用
```

### 1.2 回答
在 `langchain.schema.messages`（即 `langchain_core.messages`）子包中，`BaseMessage`、`BaseMessageChunk` 和 `BaseDataContentBlock` 是实现 LangChain 消息传递机制的核心抽象类。它们语义上的职责可以从**消息生命周期**、**消息的结构层级**、以及**多模态扩展性**这几个维度来理解：

---

#### 🔹 1. `BaseMessage` — 消息的抽象表示（完整体）

**语义作用：**

* 表示一次完整的“消息”，可以来自人类（`HumanMessage`）、AI（`AIMessage`）、系统（`SystemMessage`）或函数（`FunctionMessage`）等。
* 是对 LLM 输入输出中**单个消息单元的统一抽象**，具备发送方类型、内容和附加字段。

**关键字段：**

```python
class BaseMessage:
    content: Union[str, List[BaseContentBlock]]
    type: str  # "human", "ai", "system", etc.
    name: Optional[str]  # 发送者名称，函数调用时有用
```

**语义比喻：**
像是一次完整的发言，包含说话的人、说的内容，甚至可以带图像、函数结果等结构化数据。

---

#### 🔹 2. `BaseMessageChunk` — 消息的“部分片段”表示

**语义作用：**

* 表示一个消息的**部分内容**，尤其用于 LLM 的**流式生成（streaming）**。
* 在多轮对话生成时，LLM 通常按 token 分批返回，每一批可以被包装成一个 `BaseMessageChunk`。
* 多个 `BaseMessageChunk` 可以拼接（`+` 操作）成完整的 `BaseMessage`。

**关键字段：**

```python
class BaseMessageChunk:
    content: Union[str, List[BaseContentBlock]]
    type: str
```

**语义比喻：**
你在打字发消息时，对方逐字符看到你输入的内容，这些字符片段就是 `BaseMessageChunk`，最终组成完整的 `BaseMessage`。

---

#### 🔹 3. `BaseDataContentBlock` — 消息中非文本内容的基本构件（多模态支持）

**语义作用：**

* 用于构建 `BaseMessage` 中的非字符串内容，比如图像、表格、音频、代码块等。
* `BaseMessage.content` 不仅可以是字符串，还可以是 `List[BaseContentBlock]`，这些块中就包含了 `BaseDataContentBlock`。
* 支撑多模态场景（例如 Vision 模型、Code Interpreter）。

**例子：**

* `ImageContentBlock` 继承自 `BaseDataContentBlock`
* `TextContentBlock` 继承自 `BaseContentBlock`

**语义比喻：**
如果 `BaseMessage` 是一封邮件，`BaseDataContentBlock` 就是附件，比如图像、Excel 表格、PDF 文件等。

---

#### 🧩 三者关系总结：

| 类名                     | 表示什么？         | 使用场景                 | 是否完整消息？   |
| ---------------------- | ------------- | -------------------- | --------- |
| `BaseMessage`          | 一条完整消息        | 对话上下文、完整发言、非流式输入/输出  | ✅         |
| `BaseMessageChunk`     | 一条消息的部分内容（分片） | LLM 流式输出、token 一块块生成 | ❌         |
| `BaseDataContentBlock` | 非文本的消息块内容     | 多模态内容嵌入（图像、音频、表格等）   | ❌（消息内容片段） |

---

所以我们先来看 BaseMessage


## 2. BaseMessage
```python
class BaseMessage(Serializable):
    """Base abstract message class.

    Messages are the inputs and outputs of ChatModels.
    """

    content: Union[str, list[Union[str, dict]]]
    """The string contents of the message."""

    additional_kwargs: dict = Field(default_factory=dict)
    """Reserved for additional payload data associated with the message.

    For example, for a message from an AI, this could include tool calls as
    encoded by the model provider.
    """

    response_metadata: dict = Field(default_factory=dict)
    """Response metadata. For example: response headers, logprobs, token counts, model
    name."""

    type: str
    """The type of the message. Must be a string that is unique to the message type.

    The purpose of this field is to allow for easy identification of the message type
    when deserializing messages.
    """

    name: Optional[str] = None
    """An optional name for the message.

    This can be used to provide a human-readable name for the message.

    Usage of this field is optional, and whether it's used or not is up to the
    model implementation.
    """

    id: Optional[str] = Field(default=None, coerce_numbers_to_str=True)
    """An optional unique identifier for the message. This should ideally be
    provided by the provider/model which created the message."""

```


| 属性名                 | 类型                                   | 含义                                                       | 使用场景 / 示例                                                                 |
| ------------------- | ------------------------------------ | -------------------------------------------------------- | ------------------------------------------------------------------------- |
| `content`           | `Union[str, list[Union[str, dict]]]` | 消息的**主要内容**。可以是字符串，也可以是字符串和结构化内容（如图片）的混合列表。              | 对话中用户/AI 说的那句话。例如 `"Hello"`，或图文混合的 `[{"type": "image_url", ...}, "描述文字"]` |
| `additional_kwargs` | `dict`（默认空）                          | **附加参数**，模型提供方可能用它传递额外信息，如函数调用参数、工具调用格式等。                | OpenAI 工具调用中，`tool_calls` 数据就会放在这里                                        |
| `response_metadata` | `dict`（默认空）                          | **模型响应元信息**，如 `logprobs`、token 数、响应头、模型名称等。              | 日志记录、评估、计费分析时读取                                                           |
| `type`              | `str`                                | 表示**消息类型**，如 `"human"`、`"ai"`、`"system"`、`"function"` 等。 | 用于识别消息是由谁发送的、分类用途                                                         |
| `name`              | `Optional[str]`（默认 None）             | 可选的**发送者名称**，多用于 function call 中标识工具名称，或多角色对话时区分说话人。     | 如 `"function": {"name": "search_tools"}`                                  |
| `id`                | `Optional[str]`（默认 None）             | 可选的**消息唯一 ID**，由提供方或开发者设置。                               | 用于消息追踪、日志标识、去重等用途                                                         |



