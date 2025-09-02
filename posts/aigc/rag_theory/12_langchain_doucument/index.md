# Langchain Rag 抽象详解


上一节我们了解了 Langchain 中有关 RAG 的抽象。这一节我们来学习这些抽象的详细定义。

## 1. Document
```bash
BaseMedia
 └── Blob           ← 原始二进制数据
Document           ← 可用于 LLM 的文档
BaseDocumentTransformer ← 文档的处理/转换器
```

- BaseMedia / Blob 强调 原始内容
- Document 强调 可处理、可索引、可喂给 LLM 的内容
- BaseDocumentTransformer 强调 对文档的加工或转换逻辑

### 1.1 `Document`

```python
class BaseMedia(Serializable):


    # The ID field is optional at the moment.
    # It will likely become required in a future major release after
    # it has been adopted by enough vectorstore implementations.
    id: Optional[str] = Field(default=None, coerce_numbers_to_str=True)
    """An optional identifier for the document.

    Ideally this should be unique across the document collection and formatted
    as a UUID, but this will not be enforced.

    .. versionadded:: 0.2.11
    """

    metadata: dict = Field(default_factory=dict)


class Blob(BaseMedia):
    data: Union[bytes, str, None] = None
    """Raw data associated with the blob."""
    mimetype: Optional[str] = None
    """MimeType not to be confused with a file extension."""
    encoding: str = "utf-8"
    """Encoding to use if decoding the bytes into a string.

    Use utf-8 as default encoding, if decoding to string.
    """
    path: Optional[PathLike] = None
    """Location where the original content was found."""

    model_config = ConfigDict(
        arbitrary_types_allowed=True,
        frozen=True,
    )


class Document(BaseMedia):
    page_content: str
    """String text."""
    type: Literal["Document"] = "Document"
```


比较复杂的是 Blob，下面是其**属性** 和 **方法**的说明

#### `Blob` 的属性及作用

| 属性名            | 类型                        | 作用                                                          |
| -------------- | ------------------------- | ----------------------------------------------------------- |
| `data`         | `Union[bytes, str, None]` | 存放原始数据，可以是字符串、二进制或为空（如果是从文件路径懒加载）。                          |
| `mimetype`     | `Optional[str]`           | 数据的 **MIME 类型**（如 `text/plain`、`application/pdf`），用于标识内容格式。 |
| `encoding`     | `str`（默认 `"utf-8"`）       | 指定在字符串和二进制之间转换时使用的编码。                                       |
| `path`         | `Optional[PathLike]`      | 如果数据来自文件，这里保存文件路径。                                          |
| `metadata`     | `dict`（继承自 `BaseMedia`）   | 存放与数据相关的额外信息，例如 `source`。                                   |
| `model_config` | `ConfigDict`              | Pydantic 配置：允许任意类型字段，冻结实例避免修改。                              |
| `source`（属性方法） | `Optional[str]`           | 数据来源（优先使用 `metadata["source"]`，否则用 `path`）。                 |

---

#### `Blob` 的方法及作用

| 方法名                               | 作用                                              |
| --------------------------------- | ----------------------------------------------- |
| `check_blob_is_valid` (validator) | 在实例化前验证：必须提供 `data` 或 `path`，否则报错。              |
| `as_string()`                     | 将数据以 **字符串** 形式返回。如果是二进制则解码；如果是路径则读取文件。         |
| `as_bytes()`                      | 将数据以 **字节** 形式返回。如果是字符串则编码；如果是路径则读取文件。          |
| `as_bytes_io()`                   | 将数据以 **字节流（IO 对象）** 的形式返回，可用于需要文件流的场景（如上传 API）。 |
| `from_path(path, ...)`            | 工厂方法：从文件路径创建 `Blob`，默认不立即加载数据，而是只保存路径。          |
| `from_data(data, ...)`            | 工厂方法：直接从内存数据（字符串/字节）创建 `Blob`。                  |
| `__repr__()`                      | 定义对象的字符串表示形式，打印时显示 `Blob id` 和来源。               |


### 1.2. `BaseDocumentTransformer`
BaseDocumentTransformer 只有一个抽象方法 `transform_documents`，用于对文档进行转换。

```python
class BaseDocumentTransformer(ABC):
    @abstractmethod
    def transform_documents(
        self, documents: Sequence[Document], **kwargs: Any
    ) -> Sequence[Document]:

    async def atransform_documents(
        self, documents: Sequence[Document], **kwargs: Any
    ) -> Sequence[Document]:
        return await run_in_executor(
            None, self.transform_documents, documents, **kwargs
        )
```

### 1.3 BaseDocumentTransformer 具体实现

BaseDocumentTransformer 的具体实现都在 langchain_text_splitters 这个包内。

![UML 类图](/images/langchain/splitters.svg)


## 2. `DocumentLoader`

```bash
BaseLoader                ← 抽象接口，定义 load() 方法
 ├── BlobLoader            ← 加载原始二进制数据
 └── LangSmithLoader       ← 从 LangSmith 平台加载数据

BaseBlobParser             ← 将 Blob 解析成 Document
```

### 2.1 `BaseLoader`
BaseLoader 定义 `lazy_load` 抽象方法，并存在如下调用链:

```bash
load
    lazy_load
```

BaseLoader 还提供了一个 load_and_split，用于将 Documents 分成多个 chunk。

```python
class BaseLoader(ABC): 
    def load(self) -> list[Document]:
        """Load data into Document objects."""
        return list(self.lazy_load())

    async def aload(self) -> list[Document]:
        """Load data into Document objects."""
        return [document async for document in self.alazy_load()]

    def lazy_load(self) -> Iterator[Document]:
        """A lazy loader for Documents."""
        if type(self).load != BaseLoader.load:
            return iter(self.load())
        msg = f"{self.__class__.__name__} does not implement lazy_load()"
        raise NotImplementedError(msg)

    async def alazy_load(self) -> AsyncIterator[Document]:
        """A lazy loader for Documents."""
        iterator = await run_in_executor(None, self.lazy_load)
        done = object()
        while True:
            doc = await run_in_executor(None, next, iterator, done)
            if doc is done:
                break
            yield doc  # type: ignore[misc]


    def load_and_split(
        self, text_splitter: Optional[TextSplitter] = None
    ) -> list[Document]:
        """Load Documents and split into chunks. Chunks are returned as Documents.

        Do not override this method. It should be considered to be deprecated!

        Args:
            text_splitter: TextSplitter instance to use for splitting documents.
              Defaults to RecursiveCharacterTextSplitter.

        Returns:
            List of Documents.
        """
        if text_splitter is None:
            try:
                from langchain_text_splitters import RecursiveCharacterTextSplitter
            except ImportError as e:
                msg = (
                    "Unable to import from langchain_text_splitters. Please specify "
                    "text_splitter or install langchain_text_splitters with "
                    "`pip install -U langchain-text-splitters`."
                )
                raise ImportError(msg) from e

            text_splitter_: TextSplitter = RecursiveCharacterTextSplitter()
        else:
            text_splitter_ = text_splitter
        docs = self.load()
        return text_splitter_.split_documents(docs)
```

### 2.2 `BlobLoader`
BlobLoader 定义 `yield_blobs` 抽象方法，用于返回一个 Blob 迭代器。
BaseBlobParser 定义 `lazy_parse` 抽象方法，用于将 Blob 解析成 Document。

BaseBlobParser 中存在如下调用链:

```bash
parse
    lazy_parse
```

```python
class BlobLoader(ABC):
    """Abstract interface for blob loaders implementation.

    Implementer should be able to load raw content from a storage system according
    to some criteria and return the raw content lazily as a stream of blobs.
    """

    @abstractmethod
    def yield_blobs(
        self,
    ) -> Iterable[Blob]:
        """A lazy loader for raw data represented by LangChain's Blob object.

        Returns:
            A generator over blobs
        """


class BaseBlobParser(ABC):
    @abstractmethod
    def lazy_parse(self, blob: Blob) -> Iterator[Document]:

    def parse(self, blob: Blob) -> list[Document]:
        return list(self.lazy_parse(blob))
```

### 2.3 BaseLoader 具体实现
BaseLoader 的具体实现位于 langchain_unstructured、langchain_community.document_loaders 中。这两个库底层都使用了 [Unstructured](https://github.com/Unstructured-IO/unstructured)

这三个库的关系有点绕，后面我们会有一节专门介绍 unstructured 的使用。

#### 在 RAG 流程中的位置

```text
原始文件 (PDF/Word/HTML/图片)
   │
   ▼
langchain_unstructured (Document Loaders)
   │
   ▼
标准 Document 对象
   │
   ▼
TextSplitter → Embeddings → VectorStore → Retriever → LLM
```

## 3. embedding
```
Embeddings                     ← 抽象接口：文本 → 向量
 └── FakeEmbeddings            ← 测试用伪向量
      └── DeterministicFakeEmbedding  ← 可重复伪向量
```

- Embeddings：真实或通用向量化接口
- FakeEmbeddings：占位或随机向量，用于测试
- DeterministicFakeEmbedding：可重复向量，用于可靠测试

### 3.1 Embeddings
Embeddings 定义了两个抽象方法:
1. embed_documents: 用于将文档列表转换为向量列表
2. embed_query: 用于将查询文本转换为向量


```python
class Embeddings(ABC):
    @abstractmethod
    def embed_documents(self, texts: list[str]) -> list[list[float]]:
        """Embed search docs.

        Args:
            texts: List of text to embed.

        Returns:
            List of embeddings.
        """

    @abstractmethod
    def embed_query(self, text: str) -> list[float]:
        """Embed query text.

        Args:
            text: Text to embed.

        Returns:
            Embedding.
        """

    async def aembed_documents(self, texts: list[str]) -> list[list[float]]:
        """Asynchronous Embed search docs.

        Args:
            texts: List of text to embed.

        Returns:
            List of embeddings.
        """
        return await run_in_executor(None, self.embed_documents, texts)

    async def aembed_query(self, text: str) -> list[float]:
        """Asynchronous Embed query text.

        Args:
            text: Text to embed.

        Returns:
            Embedding.
        """
        return await run_in_executor(None, self.embed_query, text)
```

### 3.2 Embedding 具体实现
1. 开源模型的 Embedding 位于 `langchain_community.embeddings`
2. 闭源服务有各自独立的包:
    - langchain_openai.embeddings → OpenAIEmbeddings
    - langchain_google_genai.embeddings → Google Gemini 的 embedding

## 4. Retriever
`BaseRetriever` 是整个检索器模块的基础抽象：

```
BaseRetriever
 ├─ 向量数据库检索器（VectorStoreRetriever）
 ├─ 关键词检索器（KeywordRetriever）
 └─ 其他自定义检索器
```

* 抽象了“如何从文档集合中找到相关信息”的通用接口
* 提供统一调用方式给 LLM 或上层应用

### 4.1 BaseRetriever

BaseRetriever 定义了抽象接口:
1. _get_relevant_documents，类似的 get_relevant_documents 已经标注为 deprecated
2. _aget_relevant_documents，类似的 aget_relevant_documents 已经标注为 deprecated

invoke 方法会调用 _get_relevant_documents 

```basg
invoke
    _get_relevant_documents

ainvoke
    _aget_relevant_documents，类似的
```

```python
class BaseRetriever(RunnableSerializable[RetrieverInput, RetrieverOutput], ABC):
    model_config = ConfigDict(
        arbitrary_types_allowed=True,
    )

    _new_arg_supported: bool = False
    _expects_other_args: bool = False
    tags: Optional[list[str]] = None
    """Optional list of tags associated with the retriever. Defaults to None.
    These tags will be associated with each call to this retriever,
    and passed as arguments to the handlers defined in `callbacks`.
    You can use these to eg identify a specific instance of a retriever with its
    use case.
    """
    metadata: Optional[dict[str, Any]] = None
    """Optional metadata associated with the retriever. Defaults to None.
    This metadata will be associated with each call to this retriever,
    and passed as arguments to the handlers defined in `callbacks`.
    You can use these to eg identify a specific instance of a retriever with its
    use case.
    """

    @abstractmethod
    def _get_relevant_documents(
        self, query: str, *, run_manager: CallbackManagerForRetrieverRun
    ) -> list[Document]:
        pass

    async def _aget_relevant_documents(
        self, query: str, *, run_manager: AsyncCallbackManagerForRetrieverRun
    ) -> list[Document]:
        return await run_in_executor(
            None,
            self._get_relevant_documents,
            query,
            run_manager=run_manager.get_sync(),
        )
        pass

    @override
    def invoke(
        self, input: str, config: Optional[RunnableConfig] = None, **kwargs: Any
    ) -> list[Document]:
        """Invoke the retriever to get relevant documents.

        Main entry point for synchronous retriever invocations.

        Args:
            input: The query string.
            config: Configuration for the retriever. Defaults to None.
            kwargs: Additional arguments to pass to the retriever.
        """
        from langchain_core.callbacks.manager import CallbackManager

        config = ensure_config(config)
        inheritable_metadata = {
            **(config.get("metadata") or {}),
            **self._get_ls_params(**kwargs),
        }
        callback_manager = CallbackManager.configure(
            config.get("callbacks"),
            None,
            verbose=kwargs.get("verbose", False),
            inheritable_tags=config.get("tags"),
            # tag 会被传入 callback
            local_tags=self.tags,
            inheritable_metadata=inheritable_metadata,
            # metadata 会被传入 callback
            local_metadata=self.metadata,
        )
        run_manager = callback_manager.on_retriever_start(
            None,
            input,
            name=config.get("run_name") or self.get_name(),
            run_id=kwargs.pop("run_id", None),
        )
        try:
            # _expects_other_args 为 True 时，kwargs 会被传入 _get_relevant_documents
            kwargs_ = kwargs if self._expects_other_args else {}
            if self._new_arg_supported:
                # 是否传入 run_manager 取决于 _new_arg_supported
                result = self._get_relevant_documents(
                    input, run_manager=run_manager, **kwargs_
                )
            else:
                result = self._get_relevant_documents(input, **kwargs_)
        except Exception as e:
            run_manager.on_retriever_error(e)
            raise
        else:
            run_manager.on_retriever_end(
                result,
            )
            return result
```

### 4.2 BaseRetriever 具体实现
BaseRetriever 的具体实现，主要是下面要介绍的 VectorStoreRetriever。

## 5. VectorStore
```
VectorStore                  ← 向量数据库抽象接口
 └── InMemoryVectorStore      ← 内存实现版本（简单/测试用）

VectorStoreRetriever          ← Retriever 封装，调用 VectorStore 返回 Document
```

* `VectorStore`：定义向量数据库接口
* `VectorStoreRetriever`：基于向量数据的检索器实现

### 5.1 VectorStore
VectorStore 提供的方法比较多，这里我们忽略异步方法。其提供了如下抽象方法:
1. add_texts: 向向量数据添加文档，可以借助 add_documents 实现
2. add_documents 向向量数据添加文档，可以借助 add_texts 实现，这两个方法任意实现其一即可
3. embeddings: 可选实现 embedding
4. delete: 通过 id 删除文档
5. get_by_ids：通过 id 获取文档
6. search: 查询相似文档

明白了，我将删除参数列，并且凡是直接抛出 `NotImplementedError` 的方法都标注为“抽象方法”。整理后的表格如下：

| 方法名                                        | 类型   | 简要说明                                                         |
| ------------------------------------------ | ---- | ------------------------------------------------------------ |
| `add_texts`                                | 抽象方法 | 将文本通过 embeddings 添加到向量存储中，返回生成的 ID 列表，可以借助 add_documents 实现                        |
| `add_documents`                            | 普通方法 | 将 Document 对象添加或更新到向量存储中，可以借助 add_texts 实现                   |
| `embeddings`                               | 属性   | 返回当前 VectorStore 的 embedding 对象（如果实现了）                       |
| `delete`                                   | 抽象方法 | 删除指定 ID 的文档，需子类实现                                            |
| `get_by_ids`                               | 抽象方法 | 根据 ID 获取文档，需子类实现                                             |
| `search`                                   | 普通方法 | 根据指定搜索类型（similarity、mmr、similarity\_score\_threshold）返回最相似文档 |
| `similarity_search`                        | 抽象方法 | 返回最相似文档列表，子类必须实现                                             |
| `similarity_search_with_relevance_scores`  | 普通方法 | 返回文档及相关性分数，可根据 `score_threshold` 过滤                          |
| `_similarity_search_with_relevance_scores` | 普通方法 | 默认实现的带相关性分数的相似性搜索，可被子类修改                                     |
| `_select_relevance_score_fn`               | 抽象方法 | 返回用于计算相关性分数的函数，需子类实现                                         |
| `similarity_search_with_score`             | 抽象方法 | 根据距离或分数返回文档及相似度分数，需子类实现                                      |
| `max_marginal_relevance_search`            | 抽象方法 | 最大边际相关性搜索，返回兼顾相似性和多样性的文档，需子类实现                               |
| `similarity_search_by_vector`              | 抽象方法 | 根据向量返回最相似文档，需子类实现                                            |
| `max_marginal_relevance_search_by_vector`  | 抽象方法 | 基于向量的最大边际相关性搜索，需子类实现                                         |
| `from_documents`                           | 类方法  | 使用文档和 embedding 初始化 VectorStore                              |
| `from_texts`                               | 抽象方法 | 使用文本和 embedding 初始化 VectorStore，子类必须实现                       |
| `as_retriever`                             | 普通方法 | 返回一个基于该 VectorStore 的 `VectorStoreRetriever` 对象              |


VectorStore 中包含如下调用链:

```bash
add_texts
    add_documents
add_documents
    add_texts

search
    # if search_type == "similarity":
    similarity_search

    # if search_type == "similarity_score_threshold":
    similarity_search_with_relevance_scores
        _similarity_search_with_relevance_scores
            _select_relevance_score_fn
            similarity_search_with_score

    # if search_type == "mmr":
    max_marginal_relevance_search


from_documents
    from_texts
```



```python
class VectorStore(ABC):
    """Interface for vector store."""
    def add_texts(
        self,
        texts: Iterable[str],
        metadatas: Optional[list[dict]] = None,
        *,
        ids: Optional[list[str]] = None,
        **kwargs: Any,
    ) -> list[str]:
        """Run more texts through the embeddings and add to the vectorstore."""
        pass

    def add_documents(self, documents: list[Document], **kwargs: Any) -> list[str]:
        """Add or update documents in the vectorstore.
        """
        pass

    @property
    def embeddings(self) -> Optional[Embeddings]:
        """Access the query embedding object if available."""
        logger.debug(
            "The embeddings property has not been implemented for %s",
            self.__class__.__name__,
        )
        return None

    def delete(self, ids: Optional[list[str]] = None, **kwargs: Any) -> Optional[bool]:
        msg = "delete method must be implemented by subclass."
        raise NotImplementedError(msg)

    def get_by_ids(self, ids: Sequence[str], /) -> list[Document]:
        """Get documents by their IDs."""
        msg = f"{self.__class__.__name__} does not yet support get_by_ids."
        raise NotImplementedError(msg)

    def add_documents(self, documents: list[Document], **kwargs: Any) -> list[str]:
        """Add or update documents in the vectorstore.
        """
        if type(self).add_texts != VectorStore.add_texts:
            if "ids" not in kwargs:
                ids = [doc.id for doc in documents]

                # If there's at least one valid ID, we'll assume that IDs
                # should be used.
                if any(ids):
                    kwargs["ids"] = ids

            texts = [doc.page_content for doc in documents]
            metadatas = [doc.metadata for doc in documents]
            return self.add_texts(texts, metadatas, **kwargs)
        msg = (
            f"`add_documents` and `add_texts` has not been implemented "
            f"for {self.__class__.__name__} "
        )
        raise NotImplementedError(msg)

    def search(self, query: str, search_type: str, **kwargs: Any) -> list[Document]:
        """Return docs most similar to query using a specified search type.

        Args:
            query: Input text
            search_type: Type of search to perform. Can be "similarity",
                "mmr", or "similarity_score_threshold".
            **kwargs: Arguments to pass to the search method.

        """
        if search_type == "similarity":
            return self.similarity_search(query, **kwargs)
        if search_type == "similarity_score_threshold":
            docs_and_similarities = self.similarity_search_with_relevance_scores(
                query, **kwargs
            )
            return [doc for doc, _ in docs_and_similarities]
        if search_type == "mmr":
            return self.max_marginal_relevance_search(query, **kwargs)
        msg = (
            f"search_type of {search_type} not allowed. Expected "
            "search_type to be 'similarity', 'similarity_score_threshold'"
            " or 'mmr'."
        )
        raise ValueError(msg)

    @abstractmethod
    def similarity_search(
        self, query: str, k: int = 4, **kwargs: Any
    ) -> list[Document]:
        """Return docs most similar to query.

        Args:
            query: Input text.
            k: Number of Documents to return. Defaults to 4.
            **kwargs: Arguments to pass to the search method.

        Returns:
            List of Documents most similar to the query.
        """

    def similarity_search_with_relevance_scores(
        self,
        query: str,
        k: int = 4,
        **kwargs: Any,
    ) -> list[tuple[Document, float]]:
        """Return docs and relevance scores in the range [0, 1].

        0 is dissimilar, 1 is most similar.

        Args:
            query: Input text.
            k: Number of Documents to return. Defaults to 4.
            **kwargs: kwargs to be passed to similarity search. Should include:
                score_threshold: Optional, a floating point value between 0 to 1 to
                    filter the resulting set of retrieved docs.
        """
        score_threshold = kwargs.pop("score_threshold", None)

        docs_and_similarities = self._similarity_search_with_relevance_scores(
            query, k=k, **kwargs
        )
        if any(
            similarity < 0.0 or similarity > 1.0
            for _, similarity in docs_and_similarities
        ):
            warnings.warn(
                "Relevance scores must be between"
                f" 0 and 1, got {docs_and_similarities}",
                stacklevel=2,
            )

        if score_threshold is not None:
            docs_and_similarities = [
                (doc, similarity)
                for doc, similarity in docs_and_similarities
                if similarity >= score_threshold
            ]
            if len(docs_and_similarities) == 0:
                logger.warning(
                    "No relevant docs were retrieved using the "
                    "relevance score threshold %s",
                    score_threshold,
                )
        return docs_and_similarities


    def _similarity_search_with_relevance_scores(
        self,
        query: str,
        k: int = 4,
        **kwargs: Any,
    ) -> list[tuple[Document, float]]:
        """Default similarity search with relevance scores.

        Modify if necessary in subclass.
        Return docs and relevance scores in the range [0, 1].

        0 is dissimilar, 1 is most similar.

        Args:
            query: Input text.
            k: Number of Documents to return. Defaults to 4.
            **kwargs: kwargs to be passed to similarity search. Should include:
                score_threshold: Optional, a floating point value between 0 to 1 to
                    filter the resulting set of retrieved docs

        Returns:
            List of Tuples of (doc, similarity_score)
        """
        relevance_score_fn = self._select_relevance_score_fn()
        docs_and_scores = self.similarity_search_with_score(query, k, **kwargs)
        return [(doc, relevance_score_fn(score)) for doc, score in docs_and_scores]

    def _select_relevance_score_fn(self) -> Callable[[float], float]:
        """The 'correct' relevance function.

        may differ depending on a few things, including:
        - the distance / similarity metric used by the VectorStore
        - the scale of your embeddings (OpenAI's are unit normed. Many others are not!)
        - embedding dimensionality
        - etc.

        Vectorstores should define their own selection-based method of relevance.
        """
        raise NotImplementedError


    def similarity_search_with_score(
        self, *args: Any, **kwargs: Any
    ) -> list[tuple[Document, float]]:
        """Run similarity search with distance.

        Args:
            *args: Arguments to pass to the search method.
            **kwargs: Arguments to pass to the search method.

        Returns:
            List of Tuples of (doc, similarity_score).
        """
        raise NotImplementedError


    def max_marginal_relevance_search(
        self,
        query: str,
        k: int = 4,
        fetch_k: int = 20,
        lambda_mult: float = 0.5,
        **kwargs: Any,
    ) -> list[Document]:
        """Return docs selected using the maximal marginal relevance.

        Maximal marginal relevance optimizes for similarity to query AND diversity
        among selected documents.

        Args:
            query: Text to look up documents similar to.
            k: Number of Documents to return. Defaults to 4.
            fetch_k: Number of Documents to fetch to pass to MMR algorithm.
                Default is 20.
            lambda_mult: Number between 0 and 1 that determines the degree
                of diversity among the results with 0 corresponding
                to maximum diversity and 1 to minimum diversity.
                Defaults to 0.5.
            **kwargs: Arguments to pass to the search method.

        Returns:
            List of Documents selected by maximal marginal relevance.
        """
        raise NotImplementedError

    def similarity_search_by_vector(
        self, embedding: list[float], k: int = 4, **kwargs: Any
    ) -> list[Document]:
        """Return docs most similar to embedding vector.

        """
        raise NotImplementedError

    def max_marginal_relevance_search_by_vector(
        self,
        embedding: list[float],
        k: int = 4,
        fetch_k: int = 20,
        lambda_mult: float = 0.5,
        **kwargs: Any,
    ) -> list[Document]:
        """Return docs selected using the maximal marginal relevance.

        Maximal marginal relevance optimizes for similarity to query AND diversity
        among selected documents.
        """

    @classmethod
    def from_documents(
        cls,
        documents: list[Document],
        embedding: Embeddings,
        **kwargs: Any,
    ) -> Self:
        """Return VectorStore initialized from documents and embeddings.
        """
        texts = [d.page_content for d in documents]
        metadatas = [d.metadata for d in documents]

        if "ids" not in kwargs:
            ids = [doc.id for doc in documents]

            # If there's at least one valid ID, we'll assume that IDs
            # should be used.
            if any(ids):
                kwargs["ids"] = ids

        return cls.from_texts(texts, embedding, metadatas=metadatas, **kwargs)

    @classmethod
    @abstractmethod
    def from_texts(
        cls: type[VST],
        texts: list[str],
        embedding: Embeddings,
        metadatas: Optional[list[dict]] = None,
        *,
        ids: Optional[list[str]] = None,
        **kwargs: Any,
    ) -> VST:
        """Return VectorStore initialized from texts and embeddings.
        """


    def as_retriever(self, **kwargs: Any) -> VectorStoreRetriever:
        """Return VectorStoreRetriever initialized from this VectorStore.
        Returns:
            VectorStoreRetriever: Retriever class for VectorStore.
        """
        tags = kwargs.pop("tags", None) or [*self._get_retriever_tags()]
        return VectorStoreRetriever(vectorstore=self, tags=tags, **kwargs)

```

### 5.2 VectorStoreRetriever
VectorStoreRetriever 是基于 VectorStore 实现的 Retriever。

```python
class VectorStoreRetriever(BaseRetriever):
    """Base Retriever class for VectorStore."""

    vectorstore: VectorStore
    """VectorStore to use for retrieval."""
    search_type: str = "similarity"
    """Type of search to perform. Defaults to "similarity"."""
    search_kwargs: dict = Field(default_factory=dict)
    """Keyword arguments to pass to the search function."""
    allowed_search_types: ClassVar[Collection[str]] = (
        "similarity",
        "similarity_score_threshold",
        "mmr",
    )

    model_config = ConfigDict(
        arbitrary_types_allowed=True,
    )

    @override
    def _get_relevant_documents(
        self, query: str, *, run_manager: CallbackManagerForRetrieverRun, **kwargs: Any
    ) -> list[Document]:
        kwargs_ = self.search_kwargs | kwargs
        if self.search_type == s"imilarity":
            docs = self.vectorstore.similarity_search(query, **kwargs_)
        elif self.search_type == "similarity_score_threshold":
            docs_and_similarities = (
                self.vectorstore.similarity_search_with_relevance_scores(
                    query, **kwargs_
                )
            )
            docs = [doc for doc, _ in docs_and_similarities]
        elif self.search_type == "mmr":
            docs = self.vectorstore.max_marginal_relevance_search(query, **kwargs_)
        else:
            msg = f"search_type of {self.search_type} not allowed."
            raise ValueError(msg)
        return docs
```

### 5.3 VectorStore 具体实现

### 向量数据库概览

好的 👍 我帮你整理了一些常见向量数据库，对比它们的 **实现语言、适用规模、主要优势、是否支持分布式**：

| 数据库                                              | 实现语言         | 适用规模             | 主要优势                                      | 支持分布式  |
| ------------------------------------------------ | ------------ | ---------------- | ----------------------------------------- | ------ |
| **FAISS (Facebook AI)**                          | C++ / Python | 单机，百万～上亿向量       | 高性能相似度搜索，提供多种索引结构（IVF, HNSW, PQ 等），GPU 加速 | ❌（仅单机） |
| **Annoy (Spotify)**                              | C++ / Python | 单机，百万级向量         | 内存占用小，构建和查询速度快，适合推荐场景                     | ❌      |
| **hnswlib**                                      | C++ / Python | 单机，百万～上亿向量       | 高精度 HNSW 图索引，速度和召回率优秀                     | ❌      |
| **Milvus**                                       | C++ / Go     | 分布式，十亿级向量        | 全功能向量数据库，支持多索引，强大的分布式能力，云原生               | ✅      |
| **Weaviate**                                     | Go           | 分布式，十亿级向量        | 支持向量+结构化混合搜索，GraphQL/REST API，插件化向量化模块    | ✅      |
| **Pinecone**                                     | Go / Rust    | 云服务，十亿级向量        | 全托管 SaaS，免维护，全球分布，低延迟                     | ✅      |
| **Qdrant**                                       | Rust         | 分布式，十亿级向量        | 高性能 ANN（HNSW），支持向量+过滤检索，向量压缩，高性价比         | ✅      |
| **Vespa (Yahoo/Oath)**                           | Java / C++   | 分布式，十亿级向量        | 向量+关键词混合搜索，适合搜索/推荐引擎场景                    | ✅      |
| **Redis + Redis Vector Similarity (RediSearch)** | C            | 分布式，百万～十亿级（依赖集群） | 内存数据库基础，向量+结构化查询，适合低延迟应用                  | ✅      |
| **Elasticsearch / OpenSearch + kNN 插件**          | Java         | 分布式，十亿级          | 向量+全文检索混合，适合企业已有 ES 生态                    | ✅      |

📌 总结：

* **单机轻量级** → FAISS, Annoy, hnswlib（适合实验/小规模应用）。
* **分布式企业级** → Milvus, Weaviate, Qdrant, Vespa（适合大规模生产部署）。
* **托管服务** → Pinecone（免维护）。
* **生态扩展** → Redis, Elasticsearch（如果已有这些系统，可以扩展）。

