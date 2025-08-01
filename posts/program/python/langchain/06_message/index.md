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

    def text(self) -> str:
        """Get the text content of the message.

        Returns:
            The text content of the message.
        """
        pass


    def __add__(self, other: Any) -> ChatPromptTemplate:
        """Concatenate this message with another message."""
        from langchain_core.prompts.chat import ChatPromptTemplate

        prompt = ChatPromptTemplate(messages=[self])
        return prompt + other
```

### 2.1 属性

| 属性名                 | 类型                                   | 含义                                                       | 使用场景 / 示例                                                                 |
| ------------------- | ------------------------------------ | -------------------------------------------------------- | ------------------------------------------------------------------------- |
| `content`           | `Union[str, list[Union[str, dict]]]` | 消息的**主要内容**。可以是字符串，也可以是字符串和结构化内容（如图片）的混合列表。              | 对话中用户/AI 说的那句话。例如 `"Hello"`，或图文混合的 `[{"type": "image_url", ...}, "描述文字"]` |
| `additional_kwargs` | `dict`（默认空）                          | **附加参数**，模型提供方可能用它传递额外信息，如函数调用参数、工具调用格式等。                | OpenAI 工具调用中，`tool_calls` 数据就会放在这里                                        |
| `response_metadata` | `dict`（默认空）                          | **模型响应元信息**，如 `logprobs`、token 数、响应头、模型名称等。              | 日志记录、评估、计费分析时读取                                                           |
| `type`              | `str`                                | 表示**消息类型**，如 `"human"`、`"ai"`、`"system"`、`"function"` 等。 | 用于识别消息是由谁发送的、分类用途                                                         |
| `name`              | `Optional[str]`（默认 None）             | 可选的**发送者名称**，多用于 function call 中标识工具名称，或多角色对话时区分说话人。     | 如 `"function": {"name": "search_tools"}`                                  |
| `id`                | `Optional[str]`（默认 None）             | 可选的**消息唯一 ID**，由提供方或开发者设置。                               | 用于消息追踪、日志标识、去重等用途                                                         |

### 2.2 方法
BaseMessage 没有抽象方法，有一个 text 方法，用于返回消息的文本内容。还定义了消息合并的方法。消息合并是通过 ChatPromptTemplate。从这里可以看到 ChatPromptTemplate 是 Langchain 中 Prompts 和 Message 处理的入口。


## 3. BaseMessageChunk
BaseMessageChunk 只是重载了 `__add__` 方法，用于合并多个 `BaseMessageChunk`。因为 BaseMessageChunk 代表消息中的一个 chunk 所以它只能和 BaseMessageChunk 合并。


```python
    def __add__(self, other: Any) -> BaseMessageChunk:  # type: ignore[override]
        
        if isinstance(other, BaseMessageChunk):
            # If both are (subclasses of) BaseMessageChunk,
            # concat into a single BaseMessageChunk

            return self.__class__(
                id=self.id,
                type=self.type,
                content=merge_content(self.content, other.content),
                additional_kwargs=merge_dicts(
                    self.additional_kwargs, other.additional_kwargs
                ),
                response_metadata=merge_dicts(
                    self.response_metadata, other.response_metadata
                ),
            )
        if isinstance(other, list) and all(
            isinstance(o, BaseMessageChunk) for o in other
        ):
            content = merge_content(self.content, *(o.content for o in other))
            additional_kwargs = merge_dicts(
                self.additional_kwargs, *(o.additional_kwargs for o in other)
            )
            response_metadata = merge_dicts(
                self.response_metadata, *(o.response_metadata for o in other)
            )
            return self.__class__(  # type: ignore[call-arg]
                id=self.id,
                content=content,
                additional_kwargs=additional_kwargs,
                response_metadata=response_metadata,
            )
        msg = (
            'unsupported operand type(s) for +: "'
            f"{self.__class__.__name__}"
            f'" and "{other.__class__.__name__}"'
        )
        raise TypeError(msg)
```

## 4. BaseDataContentBlock
BaseDataContentBlock 定义了非文本内容数据块。

```python
class BaseDataContentBlock(TypedDict, total=False):
    """Base class for data content blocks."""

    mime_type: NotRequired[str]
    """MIME type of the content block (if needed)."""


class URLContentBlock(BaseDataContentBlock):
    """Content block for data from a URL."""

    type: Literal["image", "audio", "file"]
    """Type of the content block."""
    source_type: Literal["url"]
    """Source type (url)."""
    url: str
    """URL for data."""


class Base64ContentBlock(BaseDataContentBlock):
    """Content block for inline data from a base64 string."""

    type: Literal["image", "audio", "file"]
    """Type of the content block."""
    source_type: Literal["base64"]
    """Source type (base64)."""
    data: str
    """Data as a base64 string."""
```

### 4.1 OpenAI 的非文本数据交互接口
提问: 能不能给我介绍一下 OpenAI Chat Completions API 中有关 image content 和 data content 的接口定义


当然可以！OpenAI 的 [Chat Completions API](https://platform.openai.com/docs/api-reference/chat/create) 目前支持 **多模态输入（包括图像）**，也就是你可以在 `messages` 中发送 **包含图片或文件的 message content**。

其中的 `content` 字段现在支持两种扩展格式：

---

#### 🧾 1. `content` 字段的整体结构

在 `Chat Completions` 中，一个消息对象长这样：

```json
{
  "role": "user",
  "content": [
    {"type": "text", "text": "请分析这张图片"},
    {
      "type": "image_url",
      "image_url": {
        "url": "data:image/jpeg;base64,...",
        "detail": "auto"
      }
    }
  ]
}
```

关键在于：`content` 不再只是一个字符串，而是一个数组，每个元素可以是不同类型的内容块（text/image）。

---

#### 🎨 2. `image_url` 类型（image content）

这是你用来发送图像给模型的方式。

```json
{
  "type": "image_url",
  "image_url": {
    "url": "<image URL 或 base64>",
    "detail": "auto" | "low" | "high"
  }
}
```

| 字段                 | 类型            | 说明                                        |
| ------------------ | ------------- | ----------------------------------------- |
| `type`             | `"image_url"` | 表示该内容块是图像                                 |
| `image_url.url`    | `string`      | 图像地址，支持 URL 或 `data:image/...;base64,...` |
| `image_url.detail` | 可选，默认 `auto`  | 控制图像分辨率或处理细节，`low` = 快速粗略，`high` = 更详细分析  |


* `url` 可以是：

  * 公网可访问的 URL（HTTPS）
  * 或 `data:` 协议的 base64 图像（最多 20MB）
* 该功能仅在支持 vision 的模型中可用（如 `gpt-4-vision-preview`）


#### 📄 3. `file_data` 类型（data content）

OpenAI 也支持通过 `file_data` 发送结构化文件，如 PDF、CSV、JSON 等，适用于 Retrieval 或 code interpreter 功能。

```json
{
  "type": "file_data",
  "file_data": {
    "file_id": "file-abc123"
  }
}
```


| 字段                  | 类型            | 说明                                                                                |
| ------------------- | ------------- | --------------------------------------------------------------------------------- |
| `type`              | `"file_data"` | 表示这个内容块是一个上传的文件                                                                   |
| `file_data.file_id` | `string`      | 上传后的文件 ID，通过 [Files API](https://platform.openai.com/docs/api-reference/files) 获取 |

这个功能目前需要你先将文件通过 `files.upload` 上传到 OpenAI 的服务器，然后才能在对话中引用。


## 4. 具体的 Message
通过 UML 类图可以看到 Langchain 定义了如下 Message 类：
1. FunctionMessage
4. ToolMessage
2. HumanMessage
3. AIMessage
5. SystemMessage
6. RemoveMessage
7. ChatMessage

提问:

```bash
我正在阅读  langchain Prompts 子包的源代码，注意到 Message 定义了如下 Message 类:
1. FunctionMessage
4. ToolMessage
2. HumanMessage
3. AIMessage
5. SystemMessage
6. RemoveMessage
7. ChatMessage
请给我从语义上解释一下这些类的的作用
```

| 类名                | 角色标识           | 用途说明                                        |
| ----------------- | -------------- | ------------------------------------------- |
| `HumanMessage`    | `user`         | 表示用户输入的自然语言（类比 OpenAI 的 "role=user"）        |
| `AIMessage`       | `assistant`    | 表示 LLM 模型的输出（类比 OpenAI 的 "role=assistant"）  |
| `SystemMessage`   | `system`       | 给模型提供行为指导、语境设定                              |
| `FunctionMessage` | `function`     | 表示函数执行结果（OpenAI Function Calling 模型支持）      |
| `ToolMessage`     | `tool`         | 表示工具执行结果（OpenAI Tool Calling 中 `tool` role） |
| `ChatMessage`     | 自定义            | 表示任意自定义角色名的消息，如 `"role": "critic"`          |

Message 的代码都比较简单，唯一比较复杂的是 ToolMessage、FunctionMessage、AIMessage 他们涉及与 OpenAI 交互的细节，需要与 OpenAI 的 API 文档进行对比，理解其作用。这一部分我们放在 tools 部分详细讲解

```python
class ChatMessage(BaseMessage):
    """Message that can be assigned an arbitrary speaker (i.e. role)."""

    role: str
    """The speaker / role of the Message."""

    type: Literal["chat"] = "chat"
    """The type of the message (used during serialization). Defaults to "chat"."""


class HumanMessage(BaseMessage):
    example: bool = False
    """Use to denote that a message is part of an example conversation.

    At the moment, this is ignored by most models. Usage is discouraged.
    Defaults to False.
    """

    type: Literal["human"] = "human"
    """The type of the message (used for serialization). Defaults to "human"."""

class RemoveMessage(BaseMessage):
    """Message responsible for deleting other messages."""

    type: Literal["remove"] = "remove"
    """The type of the message (used for serialization). Defaults to "remove"."""


class SystemMessage(BaseMessage):
    type: Literal["system"] = "system"
    """The type of the message (used for serialization). Defaults to "system"."""
```

## 5. FunctionMessage

```python
class FunctionMessage(BaseMessage):
    """Message for passing the result of executing a tool back to a model.

    FunctionMessage are an older version of the ToolMessage schema, and
    do not contain the tool_call_id field.

    The tool_call_id field is used to associate the tool call request with the
    tool call response. This is useful in situations where a chat model is able
    to request multiple tool calls in parallel.
    """

    name: str
    """The name of the function that was executed."""

    type: Literal["function"] = "function"
    """The type of the message (used for serialization). Defaults to "function"."""
```

## 6. ToolMessage

```python
class ToolMessage(BaseMessage, ToolOutputMixin):
    tool_call_id: str
    """Tool call that this message is responding to."""

    type: Literal["tool"] = "tool"
    """The type of the message (used for serialization). Defaults to "tool"."""

    artifact: Any = None
    """Artifact of the Tool execution which is not meant to be sent to the model.

    Should only be specified if it is different from the message content, e.g. if only
    a subset of the full tool output is being passed as message content but the full
    output is needed in other parts of the code.

    .. versionadded:: 0.2.17
    """

    status: Literal["success", "error"] = "success"
    """Status of the tool invocation.

    .. versionadded:: 0.2.24
    """

    additional_kwargs: dict = Field(default_factory=dict, repr=False)
    """Currently inherited from BaseMessage, but not used."""
    response_metadata: dict = Field(default_factory=dict, repr=False)
    """Currently inherited from BaseMessage, but not used."""
```

提问: 能不能想写给我介绍一下 OpenAI Tool Calling 
