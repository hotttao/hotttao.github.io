# LangChain Prompts


## 1. Prompts 的 UML 类图
![Prompts 类图](/images/langchain/prompts-all.svg)

从 UML 类图可以看出，Prompts 有如下 Base 抽象类：
1. BasePromptTemplate
2. BaseChatPromptTemplate
3. BaseMessagePrompts

### 1.1 提问
让我们问问大模型这这些基类代表的语义。

```bash
我正在阅读  langchain Prompts 子包的源代码，注意到 Prompts 包内有如下 base 的抽象类:
1. BasePromptTemplate
2. BaseChatPromptTemplate
3. BaseMessagePrompts
请给我从语义上解释一下这三个类的的作用
```

### 1.2 回答
你提到的这三个类是 `langchain.prompts` 子包中的核心抽象类，它们分别定义了不同层级的提示（Prompt）结构，主要作用是 **组织和渲染最终要发送给语言模型的 Prompt 数据结构**，语义上可以这样理解：

---

#### 1. `BasePromptTemplate`

**作用：定义基本的提示模板结构，适用于传统 LLM 输入（非 Chat 模型）**

##### ✅ 语义解释：

这是所有 prompt 模板的最基础抽象类，它用于处理 "单轮输入 + 模板字符串" 的 LLM prompt。适用于普通的 LLM 接口（如 `text-davinci-003`），提示通常是纯字符串，如：

```
"Translate the following text to French: {input}"
```

##### 🧠 核心职责：

* 提供一个统一的 `.format()` 或 `.invoke()` 方法，用于将变量（如 input）插入模板中。
* 可以包含变量校验、默认值、输入规范等。
* 子类（如 `PromptTemplate`）会实现具体模板渲染逻辑。

##### 📦 示例使用场景：

```python
PromptTemplate(input_variables=["input"], template="Translate this: {input}")
```

---

#### 2. `BaseChatPromptTemplate`

**作用：定义适用于 Chat LLM（如 ChatGPT）的一组 message 的提示结构**

##### ✅ 语义解释：

它是面向 **Chat 模型输入格式** 的抽象类。Chat 模型（如 OpenAI GPT-4）接受的输入并不是一段字符串，而是一个包含多个 message 的列表，例如：

```json
[
  {"role": "system", "content": "You are a helpful assistant."},
  {"role": "user", "content": "What's the weather today?"}
]
```

##### 🧠 核心职责：

* 定义如何组合多个 `BaseMessagePromptTemplate` 为一个完整的 Chat Prompt。
* 提供 `.format_messages()` 等接口生成标准 message list。
* 可以组合不同角色（system, user, assistant）的 prompt。

##### 📦 示例使用场景：

```python
ChatPromptTemplate.from_messages([
    ("system", "You are a helpful assistant."),
    ("user", "What is {topic}?")
])
```

---

#### 3. `BaseMessagePromptTemplate`

**作用：定义单个 message（如 user/system/assistant）的模板结构**

##### ✅ 语义解释：

这是组成一个 ChatPrompt 的最小单元。一个 ChatPrompt 是由多个 message 组成的，而每个 message 都是 `BaseMessagePromptTemplate` 的实例（或子类实例）渲染而成。

##### 🧠 核心职责：

* 定义 message 的角色（role）和模板结构。
* 渲染时可以将输入变量格式化成一个具体的 `BaseMessage`（如 `HumanMessage`, `SystemMessage` 等）。

##### 📦 示例使用场景：

```python
from langchain.prompts import HumanMessagePromptTemplate

HumanMessagePromptTemplate.from_template("Tell me about {topic}")
```

这个可以和其他 message 一起被 `ChatPromptTemplate` 组合成完整的 Chat prompt。

---

#### 📌 总结对比

| 类名                          | 面向的模型类型           | 结构                 | 语义角色               |
| --------------------------- | ----------------- | ------------------ | ------------------ |
| `BasePromptTemplate`        | 非 Chat 模型（传统 LLM） | 字符串模板              | 最基础的一种 Prompt 表达形式 |
| `BaseChatPromptTemplate`    | Chat 模型（如 GPT-4）  | 多个消息组成的列表          | Chat Prompt 的容器    |
| `BaseMessagePromptTemplate` | Chat 模型中的每条消息     | 单个角色的一条 message 模板 | Chat Prompt 的组成部分  |

---

如果你想理解它们的类关系/继承结构，我也可以为你画一个简图或 UML 图。是否需要？

## 2. BasePromptTemplate
### 2.1 属性
```python
class BasePromptTemplate(
    RunnableSerializable[dict, PromptValue], ABC, Generic[FormatOutputType]
):
    """Base class for all prompt templates, returning a prompt."""

    input_variables: list[str]
    """A list of the names of the variables whose values are required as inputs to the
    prompt."""
    optional_variables: list[str] = Field(default=[])
    """optional_variables: A list of the names of the variables for placeholder
       or MessagePlaceholder that are optional. These variables are auto inferred
       from the prompt and user need not provide them."""
    input_types: typing.Dict[str, Any] = Field(default_factory=dict, exclude=True)  # noqa: UP006
    """A dictionary of the types of the variables the prompt template expects.
    If not provided, all variables are assumed to be strings."""
    output_parser: Optional[BaseOutputParser] = None
    """How to parse the output of calling an LLM on this formatted prompt."""
    partial_variables: Mapping[str, Any] = Field(default_factory=dict)
    """A dictionary of the partial variables the prompt template carries.

    Partial variables populate the template so that you don't need to
    pass them in every time you call the prompt."""
    metadata: Optional[typing.Dict[str, Any]] = None  # noqa: UP006
    """Metadata to be used for tracing."""
    tags: Optional[list[str]] = None
    """Tags to be used for tracing."""
```

```bash
下面是 BasePromptTemplate 的源码，请你以列表的形式列举每个属性的含义
```

| 属性名                      | 类型                           | 作用/含义                                                                                               |
| ------------------------ | ---------------------------- | --------------------------------------------------------------------------------------------------- |
| **`input_variables`**    | `list[str]`                  | **必须提供的变量名列表**，用户在格式化 prompt 时，必须传入这些变量。例如模板 `"Translate {input}"`，则 `input_variables = ["input"]`。 |
| **`optional_variables`** | `list[str]`（默认空）             | **可选变量名列表**，通常用于模板中含有 `{variable}` 或 `MessagePlaceholder` 占位符，但不强制用户传入。LangChain 会自动推断并允许缺省。        |
| **`input_types`**        | `dict[str, Any]`（默认空）        | 描述每个输入变量的 **类型信息（非强制）**，用于类型校验或文档生成。如果不提供，默认认为所有变量都是字符串。                                            |
| **`output_parser`**      | `Optional[BaseOutputParser]` | 一个可选的 **输出解析器对象**，用于将 LLM 输出从纯字符串解析为结构化格式，例如提取 JSON、答案段落等。                                          |
| **`partial_variables`**  | `Mapping[str, Any]`（默认空）     | 提供部分预先填充的变量（**局部绑定**），这些变量会在每次调用模板时自动注入，无需重复传入。类似于 Python 中的 `functools.partial()`。                 |
| **`metadata`**           | `Optional[dict[str, Any]]`   | 用于 **链路追踪（Tracing）** 的元信息，用户可以通过此字段记录 Prompt 相关的上下文、来源、调用标识等辅助信息。                                   |
| **`tags`**               | `Optional[list[str]]`        | 也是用于 **追踪/监控（Tracing）**，可以为 Prompt 添加标签，供后续在仪表盘、日志中筛选（如 `"debug"`, `"production"`, `"test"` 等）。     |


### 2.2 方法
BasePromptTemplate 是 Runnable 的子类，因此必须实现 invoke 方法，并要求子类实现如下的抽象方法:

```python
class BasePromptTemplate(
    RunnableSerializable[dict, PromptValue], ABC, Generic[FormatOutputType]
):
    @abstractmethod
    def format_prompt(self, **kwargs: Any) -> PromptValue:
        """Create Prompt Value.

        Args:
            kwargs: Any arguments to be passed to the prompt template.

        Returns:
            PromptValue: The output of the prompt.
        """

    @abstractmethod
    def format(self, **kwargs: Any) -> FormatOutputType:
        """Format the prompt with the inputs.

        Args:
            kwargs: Any arguments to be passed to the prompt template.

        Returns:
            A formatted string.

        Example:

        .. code-block:: python

            prompt.format(variable1="foo")
        """

    @override
    def invoke(
        self, input: dict, config: Optional[RunnableConfig] = None, **kwargs: Any
    ) -> PromptValue:
        """Invoke the prompt.

        Args:
            input: Dict, input to the prompt.
            config: RunnableConfig, configuration for the prompt.

        Returns:
            PromptValue: The output of the prompt.
        """
        config = ensure_config(config)
        if self.metadata:
            config["metadata"] = {**config["metadata"], **self.metadata}
        if self.tags:
            config["tags"] = config["tags"] + self.tags
        return self._call_with_config(
            self._format_prompt_with_error_handling,
            input,
            config,
            run_type="prompt",
            serialized=self._serialized,
        )

    def _format_prompt_with_error_handling(self, inner_input: dict) -> PromptValue:
        inner_input_ = self._validate_input(inner_input)
        return self.format_prompt(**inner_input_)
```

`_call_with_config` 是 Runnable 提供的方法，前面我们介绍过，内部调用的是传入的 `_format_prompt_with_error_handling` 方法。

### 2.3 BaseChatPromptTemplate

BaseChatPromptTemplate 本质上是对 BasePromptTemplate 接口做了一个转换。把所有的实现转换到 format_messages 方法上。

```python
class BaseChatPromptTemplate(BasePromptTemplate, ABC):
    """Base class for chat prompt templates."""

    def format_prompt(self, **kwargs: Any) -> ChatPromptValue:
        """Format prompt. Should return a ChatPromptValue.

        Args:
            **kwargs: Keyword arguments to use for formatting.

        Returns:
            ChatPromptValue.
        """
        messages = self.format_messages(**kwargs)
        return ChatPromptValue(messages=messages)

    def format(self, **kwargs: Any) -> str:
        """Format the chat template into a string.

        Args:
            **kwargs: keyword arguments to use for filling in template variables
                      in all the template messages in this chat template.

        Returns:
            formatted string.
        """
        return self.format_prompt(**kwargs).to_string()

    @abstractmethod
    def format_messages(self, **kwargs: Any) -> list[BaseMessage]:
        """Format kwargs into a list of messages."""
```

### 2.3 StringPromptTemplate 和 PromptTemplate
所有的 Template 最终都是为了输出文本，所以我们先来看 StringPromptTemplate 和 PromptTemplate。

```python
# 也是一个抽象基类类型
class StringPromptTemplate(BasePromptTemplate, ABC):
    """String prompt that exposes the format method, returning a prompt."""

    def format_prompt(self, **kwargs: Any) -> PromptValue:
        """Format the prompt with the inputs.

        Args:
            kwargs: Any arguments to be passed to the prompt template.

        Returns:
            A formatted string.
        """
        return StringPromptValue(text=self.format(**kwargs))

class PromptTemplate(StringPromptTemplate):
    template: str
    """The prompt template."""

    template_format: PromptTemplateFormat = "f-string"
    """The format of the prompt template.
    Options are: 'f-string', 'mustache', 'jinja2'."""

    validate_template: bool = False
    """Whether or not to try validating the template."""

    def format(self, **kwargs: Any) -> str:
        """Format the prompt with the inputs.

        Args:
            kwargs: Any arguments to be passed to the prompt template.

        Returns:
            A formatted string.
        """
        kwargs = self._merge_partial_and_user_variables(**kwargs)
        return DEFAULT_FORMATTER_MAPPING[self.template_format](self.template, **kwargs)


    @classmethod
    def from_template(
        cls,
        template: str,
        *,
        template_format: PromptTemplateFormat = "f-string",
        partial_variables: Optional[dict[str, Any]] = None,
        **kwargs: Any,
    ) -> PromptTemplate:
        pass
```

PromptTemplate 定义了一个从格式化字符串或者模板，生成 Prompt 的方法。支持 'f-string', 'mustache', 'jinja2'。并提供了一个 from_template 便携方法。

### 3. PromptValue
invoke 方法最终返回的是一个 PromptValue，PromtValue 位于 langchain_core.prompts_values。下面是这个包的 UML 类图。

![Prompts Value 类图](/images/langchain/prompts-value.svg)

```python
class PromptValue(Serializable, ABC):
    """Base abstract class for inputs to any language model.

    PromptValues can be converted to both LLM (pure text-generation) inputs and
    ChatModel inputs.
    """

    @abstractmethod
    def to_string(self) -> str:
        """Return prompt value as string."""

    @abstractmethod
    def to_messages(self) -> list[BaseMessage]:
        """Return prompt as a list of Messages."""
```

```bash
问题: langchain 中 PromptValue 的语义
回答: PromptValue 是一个包含已经格式化好的 Prompt 内容的对象，无论是字符串形式还是 message list 形式，它都能统一地表示，并用于传递给 LLM。
```

| 功能              | 描述                                                                                     |
| --------------- | -------------------------------------------------------------------------------------- |
| **统一表示最终提示内容**  | 不管是传统 prompt（字符串）还是 chat prompt（message list），最终都会包装成一个 `PromptValue` 对象               |
| **与 LLM 对接接口**  | 所有 LLM 接口（如 `invoke()`）都期望传入的是 `PromptValue` 实例，它可以被 `.to_string()` 或 `.to_messages()` |
| **封装格式转换能力**    | 提供 `.to_string()` 和 `.to_messages()` 方法，用于在字符串和消息列表之间转换                                |
| **避免直接使用原始字符串** | 提高提示模板链的抽象程度，使链条中的模块解耦，不用关心底层数据格式                                                      |

PromptValue 的两种主要子类：

| 子类                  | 用于哪种模型    | 内容形式                               | 说明                                      |
| ------------------- | --------- | ---------------------------------- | --------------------------------------- |
| `StringPromptValue` | 非 Chat 模型 | 单个字符串                              | 用于像 `text-davinci-003` 这种只接受纯文本输入的模型    |
| `ChatPromptValue`   | Chat 模型   | 一组 `BaseMessage`（如 system/user/ai） | 用于像 `gpt-4`、`gpt-3.5-turbo` 这样的 chat 模型 |


**最终我们得到这样的调用关系：Prompts -> PromptValue(包含 Message) -> LLM**

下面是 PromptValue 的子类实现:

```python
class StringPromptValue(PromptValue):
    """String prompt value."""

    text: str
    """Prompt text."""
    type: Literal["StringPromptValue"] = "StringPromptValue"

    def to_string(self) -> str:
        """Return prompt as string."""
        return self.text

    def to_messages(self) -> list[BaseMessage]:
        """Return prompt as messages."""
        return [HumanMessage(content=self.text)]

class ChatPromptValue(PromptValue):
    """Chat prompt value.

    A type of a prompt value that is built from messages.
    """

    messages: Sequence[BaseMessage]
    """List of messages."""

    def to_string(self) -> str:
        """Return prompt as string."""
        return get_buffer_string(self.messages)

    def to_messages(self) -> list[BaseMessage]:
        """Return prompt as a list of messages."""
        return list(self.messages)

class ImageURL(TypedDict, total=False):
    """Image URL."""

    detail: Literal["auto", "low", "high"]
    """Specifies the detail level of the image. Defaults to "auto".
    Can be "auto", "low", or "high"."""

    url: str
    """Either a URL of the image or the base64 encoded image data."""


class ImagePromptValue(PromptValue):
    """Image prompt value."""

    image_url: ImageURL
    """Image URL."""
    type: Literal["ImagePromptValue"] = "ImagePromptValue"

    def to_string(self) -> str:
        """Return prompt (image URL) as string."""
        return self.image_url["url"]

    def to_messages(self) -> list[BaseMessage]:
        """Return prompt (image URL) as messages."""
        return [HumanMessage(content=[cast("dict", self.image_url)])]
```

## 4. BaseMessagePromptTemplate

BaseMessagePromptTemplate 用于定义单个 message（如 user/system/assistant）的模板结构。核心抽象方法是 format_messages 实现如何产出 Message。


```python
class BaseMessagePromptTemplate(Serializable, ABC):
    """Base class for message prompt templates."""

    @property
    @abstractmethod
    def input_variables(self) -> list[str]:
        """Input variables for this prompt template.

        Returns:
            List of input variables.
        """

    @abstractmethod
    def format_messages(self, **kwargs: Any) -> list[BaseMessage]:
        """Format messages from kwargs. Should return a list of BaseMessages.

        Args:
            **kwargs: Keyword arguments to use for formatting.

        Returns:
            List of BaseMessages.
        """
```

### 4.1 BaseStringMessagePromptTemplate
BaseStringMessagePromptTemplate 的实现是有点绕的，他的实现中包含了 StringPromptTemplate。但是本质上他与 StringPromptTemplate 没有任何关系，**只是复用了 StringPromptTemplate 从结构化字符串输出 String 的能力**。从语义上他抽象了如何输出一个 String Message。


```python
class BaseStringMessagePromptTemplate(BaseMessagePromptTemplate, ABC):
    """Base class for message prompt templates that use a string prompt template."""

    prompt: StringPromptTemplate
    """String prompt template."""
    additional_kwargs: dict = Field(default_factory=dict)
    """Additional keyword arguments to pass to the prompt template."""

    @abstractmethod
    def format(self, **kwargs: Any) -> BaseMessage:
        """Format the prompt template.

        Args:
            **kwargs: Keyword arguments to use for formatting.

        Returns:
            Formatted message.
        """

    def format_messages(self, **kwargs: Any) -> list[BaseMessage]:
        """Format messages from kwargs.

        Args:
            **kwargs: Keyword arguments to use for formatting.

        Returns:
            List of BaseMessages.
        """
        return [self.format(**kwargs)]

    @classmethod
    def from_template(
        cls,
        template: str,
        template_format: PromptTemplateFormat = "f-string",
        partial_variables: Optional[dict[str, Any]] = None,
        **kwargs: Any,
    ) -> Self:
        """Create a class from a string template.

        Args:
            template: a template.
            template_format: format of the template. Defaults to "f-string".
            partial_variables: A dictionary of variables that can be used to partially
                               fill in the template. For example, if the template is
                              `"{variable1} {variable2}"`, and `partial_variables` is
                              `{"variable1": "foo"}`, then the final prompt will be
                              `"foo {variable2}"`.
                              Defaults to None.
            **kwargs: keyword arguments to pass to the constructor.

        Returns:
            A new instance of this class.
        """
        prompt = PromptTemplate.from_template(
            template,
            template_format=template_format,
            partial_variables=partial_variables,
        )
        return cls(prompt=prompt, **kwargs)
```

### 4.2 ChatMessagePromptTemplate
ChatMessagePromptTemplate 定义了输出 ChatMessage 的方法。

```python
class ChatMessagePromptTemplate(BaseStringMessagePromptTemplate):
    """Chat message prompt template."""

    role: str
    """Role of the message."""

    def format(self, **kwargs: Any) -> BaseMessage:
        """Format the prompt template.

        Args:
            **kwargs: Keyword arguments to use for formatting.

        Returns:
            Formatted message.
        """
        text = self.prompt.format(**kwargs)
        return ChatMessage(
            content=text, role=self.role, additional_kwargs=self.additional_kwargs
        )
```

### 4.3 Human/AI/System Message
说实话，这里的实现我感觉真不太好，把所有的逻辑都放到了 `_StringImageMessagePromptTemplate`。但是这个类的实现里面确夹杂着各种类型的判断。这么写的目的，是为了解析多个不同格式 template 的组合。就像 prompt 类型注解所标注的。具体的 format 和 from_template 代码不在这里展示了。

```python
class _StringImageMessagePromptTemplate(BaseMessagePromptTemplate):
    """Human message prompt template. This is a message sent from the user."""

    prompt: Union[
        StringPromptTemplate,
        list[Union[StringPromptTemplate, ImagePromptTemplate, DictPromptTemplate]],
    ]
    """Prompt template."""
    additional_kwargs: dict = Field(default_factory=dict)
    """Additional keyword arguments to pass to the prompt template."""

    _msg_class: type[BaseMessage]

    @classmethod
    def from_template(
        cls: type[Self],
        template: Union[
            str,
            list[Union[str, _TextTemplateParam, _ImageTemplateParam, dict[str, Any]]],
        ],
        template_format: PromptTemplateFormat = "f-string",
        *,
        partial_variables: Optional[dict[str, Any]] = None,
        **kwargs: Any,
    ) -> Self:
        pass


class HumanMessagePromptTemplate(_StringImageMessagePromptTemplate):
    """Human message prompt template. This is a message sent from the user."""

    _msg_class: type[BaseMessage] = HumanMessage


class AIMessagePromptTemplate(_StringImageMessagePromptTemplate):
    """AI message prompt template. This is a message sent from the AI."""

    _msg_class: type[BaseMessage] = AIMessage


class SystemMessagePromptTemplate(_StringImageMessagePromptTemplate):
    """System message prompt template.

    This is a message that is not sent to the user.
    """

    _msg_class: type[BaseMessage] = SystemMessage
```


## 5. ChatPromptTemplate
到这里我们可以对 prompt 的实现做一个总结:
1. BasePromtTemplate 提供了基本基本抽象
2. ImagePromptTemplate、PromptTemplate 是 Prompt 生成的基本单元，提供了通过基本格式化字符串生成 Prompt 的基本能力
3. DictPromptTemplate 也是 Prompt 生成的基本单元，利用了递归处理逻辑，可以处理复杂的嵌套结构。这个代码没有粘贴出来，不难理解直接看源码即可。
4. BaseMessagePromptTemplate 抽象了如何生成一个 Message，复用了 ImagePromptTemplate、PromptTemplate、DictPromptTemplate 处理格式化字符串和嵌套生成的基本逻辑，语义上无关联。
5. Human/AI/System MessagePromptTemplate 支持生成多个多种的 Message。
6. UML 类图中还有几种 Template 没有介绍，这些 Template 在上面提到这些基础上，提供了复杂的组合逻辑，下面我们来一一介绍。

BaseChatPromptTemplate 最常用的子类是:
1. chat.ChatPromptTemplate
2. structured.StructuredPrompt

### 5.1 
MessagesPlaceholder 定义了通过输入参数解析输出多种 Message 的能力。

```python
class MessagesPlaceholder(BaseMessagePromptTemplate):
    variable_name: str
    """Name of variable to use as messages."""

    optional: bool = False
    """If True format_messages can be called with no arguments and will return an empty
        list. If False then a named argument with name `variable_name` must be passed
        in, even if the value is an empty list."""

    n_messages: Optional[PositiveInt] = None
    """Maximum number of messages to include. If None, then will include all.
    Defaults to None."""

    def format_messages(self, **kwargs: Any) -> list[BaseMessage]:
        """Format messages from kwargs.

        Args:
            **kwargs: Keyword arguments to use for formatting.

        Returns:
            List of BaseMessage.

        Raises:
            ValueError: If variable is not a list of messages.
        """
        value = (
            kwargs.get(self.variable_name, [])
            if self.optional
            else kwargs[self.variable_name]
        )
        if not isinstance(value, list):
            msg = (
                f"variable {self.variable_name} should be a list of base messages, "
                f"got {value} of type {type(value)}"
            )
            raise ValueError(msg)  # noqa: TRY004
        value = convert_to_messages(value)
        if self.n_messages:
            value = value[-self.n_messages :]
        return value

```

### 5.2 ChatPromptTemplate

ChatPromptTemplate 在初始化时支持接受多个多种格式 Message 的输入。这些输入表示为 MessageLikeRepresentation 类型，他们会在初始化时使用 `_convert_to_message_template` 将这些输入转换为 BaseMessage, BaseMessagePromptTemplate, BaseChatPromptTemplate。这个时候你就会注意到虽然 BaseMessagePromptTemplate, BaseChatPromptTemplate 达标不同的抽象，但是他们都有 format_messages 抽象方法。所以他们能 Union 为统一类型的输出。 

另外 BaseMessagePromptTemplate, BaseChatPromptTemplate 代表了所有的 Message 生成基类。这意味着 ChatPromptTemplate 可以接受上面描述的所有类型的 MessageTemplate。


```python
class ChatPromptTemplate(BaseChatPromptTemplate):
    messages: Annotated[list[MessageLike], SkipValidation()]
    """List of messages consisting of either message prompt templates or messages."""
    validate_template: bool = False
    """Whether or not to try validating the template."""

    def __init__(
        self,
        messages: Sequence[MessageLikeRepresentation],
        *,
        template_format: PromptTemplateFormat = "f-string",
        **kwargs: Any,
    ) -> None:
        messages_ = [
            _convert_to_message_template(message, template_format)
            for message in messages
        ]
        pass

        @classmethod
    
    # 只支持但字符串，默认代表 HumanMessagePromptTemplate
    def from_template(cls, template: str, **kwargs: Any) -> ChatPromptTemplate:
        """Create a chat prompt template from a template string.

        Creates a chat template consisting of a single message assumed to be from
        the human.

        Args:
            template: template string
            **kwargs: keyword arguments to pass to the constructor.

        Returns:
            A new instance of this class.
        """
        prompt_template = PromptTemplate.from_template(template, **kwargs)
        message = HumanMessagePromptTemplate(prompt=prompt_template)
        return cls.from_messages([message])

    @classmethod
    def from_messages(
        cls,
        messages: Sequence[MessageLikeRepresentation],
        template_format: PromptTemplateFormat = "f-string",
    ) -> ChatPromptTemplate:
        return cls(messages, template_format=template_format)

    
        def format_messages(self, **kwargs: Any) -> list[BaseMessage]:
        """Format the chat template into a list of finalized messages.

        Args:
            **kwargs: keyword arguments to use for filling in template variables
                      in all the template messages in this chat template.

        Returns:
            list of formatted messages.
        """
        kwargs = self._merge_partial_and_user_variables(**kwargs)
        result = []
        for message_template in self.messages:
            if isinstance(message_template, BaseMessage):
                result.extend([message_template])
            elif isinstance(
                message_template, (BaseMessagePromptTemplate, BaseChatPromptTemplate)
            ):
                message = message_template.format_messages(**kwargs)
                result.extend(message)
            else:
                msg = f"Unexpected input: {message_template}"
                raise ValueError(msg)  # noqa: TRY004
        return result


def _convert_to_message_template(
    message: MessageLikeRepresentation,
    template_format: PromptTemplateFormat = "f-string",
) -> Union[BaseMessage, BaseMessagePromptTemplate, BaseChatPromptTemplate]:
    pass


MessageLike = Union[BaseMessagePromptTemplate, BaseMessage, BaseChatPromptTemplate]

MessageLikeRepresentation = Union[
    MessageLike,
    tuple[
        Union[str, type],
        Union[str, list[dict], list[object]],
    ],
    str,
    dict[str, Any],
]
```

### 5.3 StructuredPrompt
StructuredPrompt 这个类查了半天才理解。它在 ChatPromptTemplate 的基础上增加了 schema 这个字段是要传递给 BaseLanguageModel 设置 BaseLanguageModel 的输出。这个我们后面介绍 LanguageModel 抽象时在看。

```python
@beta()
class StructuredPrompt(ChatPromptTemplate):
    """Structured prompt template for a language model."""

    schema_: Union[dict, type]
    """Schema for the structured prompt."""
    structured_output_kwargs: dict[str, Any] = Field(default_factory=dict)

    def __init__(
        self,
        messages: Sequence[MessageLikeRepresentation],
        schema_: Optional[Union[dict, type[BaseModel]]] = None,
        *,
        structured_output_kwargs: Optional[dict[str, Any]] = None,
        template_format: PromptTemplateFormat = "f-string",
        **kwargs: Any,
    ) -> None:
        schema_ = schema_ or kwargs.pop("schema")
        structured_output_kwargs = structured_output_kwargs or {}
        for k in set(kwargs).difference(get_pydantic_field_names(self.__class__)):
            structured_output_kwargs[k] = kwargs.pop(k)
        super().__init__(
            messages=messages,
            schema_=schema_,
            structured_output_kwargs=structured_output_kwargs,
            template_format=template_format,
            **kwargs,
        )

    @override
    def __or__(
        self,
        other: Union[
            Runnable[Any, Other],
            Callable[[Iterator[Any]], Iterator[Other]],
            Callable[[AsyncIterator[Any]], AsyncIterator[Other]],
            Callable[[Any], Other],
            Mapping[str, Union[Runnable[Any, Other], Callable[[Any], Other], Any]],
        ],
    ) -> RunnableSerializable[dict, Other]:
        return self.pipe(other)

    def pipe(
        self,
        *others: Union[
            Runnable[Any, Other],
            Callable[[Iterator[Any]], Iterator[Other]],
            Callable[[AsyncIterator[Any]], AsyncIterator[Other]],
            Callable[[Any], Other],
            Mapping[str, Union[Runnable[Any, Other], Callable[[Any], Other], Any]],
        ],
        name: Optional[str] = None,
    ) -> RunnableSerializable[dict, Other]:
        """Pipe the structured prompt to a language model.

        Args:
            others: The language model to pipe the structured prompt to.
            name: The name of the pipeline. Defaults to None.

        Returns:
            A RunnableSequence object.

        Raises:
            NotImplementedError: If the first element of `others`
            is not a language model.
        """
        if (others and isinstance(others[0], BaseLanguageModel)) or hasattr(
            others[0], "with_structured_output"
        ):
            return RunnableSequence(
                self,
                others[0].with_structured_output(
                    self.schema_, **self.structured_output_kwargs
                ),
                *others[1:],
                name=name,
            )
        msg = "Structured prompts need to be piped to a language model."
        raise NotImplementedError(msg)

```
