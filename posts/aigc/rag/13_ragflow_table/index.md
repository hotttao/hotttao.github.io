# RagFlow 中的表


在深入到具体业务代码之前，我们先来看看 ragflow 都定义了哪些表。

## 1. Ragflow 定义的表
Ragflow 定义了以下表，这些表位于 `api\db\db_models.py`

| 表名 | 主要字段/关联 | 作用说明 |
|---|---|---|
| user | `id,email,language,timezone` | 用户账号信息与登录属性 |
| tenant | `id,llm_id,embd_id,asr_id,img2txt_id,rerank_id` | 租户/工作区，默认模型与额度配置 |
| user_tenant | `user_id,tenant_id,role` | 用户与租户的关系与角色 |
| invitation_code | `code,user_id,tenant_id` | 邀请码与受邀记录 |
| llm_factories | `name,tags,logo` | 模型厂商目录 |
| llm | 复合主键(`fid,llm_name`), `model_type,tags,is_tools` | 厂商下具体模型清单与能力标签 |
| tenant_llm | 复合主键(`tenant_id,llm_factory,llm_name`), `api_key,api_base,max_tokens` | 租户绑定的具体模型配置（包含 API Key、Base URL、额度等） |
| tenant_langfuse | `tenant_id,host,public_key,secret_key` | 租户的 Langfuse 观测配置 |
| knowledgebase | `id,tenant_id,language,embd_id,parser_id,parser_config,pagerank` | 知识库元数据与默认解析/检索参数 |
| document | `id,kb_id,parser_id,parser_config,name,location,size,progress` | 文档元数据、解析进度与统计 |
| file | `id,parent_id,tenant_id,name,location,size,type` | 文件/文件夹元信息（资源管理） |
| file2document | `file_id,document_id` | 文件与文档映射关系 |
| task | `id,doc_id,from_page,to_page,progress,retry_count,digest,chunk_ids` | 文档解析任务与进度、重试和产物 |
| dialog | `id,tenant_id,llm_id,rerank_id,prompt_config,llm_setting,top_k,kb_ids` | 对话应用配置（检索+生成） |
| conversation | `id,dialog_id,user_id,message,reference` | 某对话应用下的会话与消息摘要 |
| api_token | 复合主键(`tenant_id,token`), `dialog_id,source,beta` | API 访问令牌（可绑定应用/来源） |
| api_4_conversation | `id,dialog_id,user_id,tokens,duration,round,errors,source` | API 调用产生的会话与用量/错误日志 |
| user_canvas | `id,user_id,title,permission,dsl` | 用户画布（编排/工作流）实例 |
| canvas_template | `id,title,canvas_type,dsl` | 画布模板库 |
| user_canvas_version | `id,user_canvas_id,title,description,dsl` | 画布版本历史 |
| mcp_server | `id,tenant_id,name,url,server_type,headers,variables` | MCP Server 注册与调用配置 |
| search | `id,tenant_id,name,search_config` | 独立搜索配置（跨 KB/Doc、检索/重排/高亮等） |

按照相关性，这些表可以分为:

### 1.1 用户与权限管理

| 表名 | 作用 |
|---|---|
| `user` | 用户账号信息与登录属性 |
| `tenant` | 租户/工作区，默认模型与额度配置 |
| `user_tenant` | 用户与租户的关系与角色 |
| `invitation_code` | 邀请码与受邀记录 |

### 1.2 模型与AI能力管理
| 表名 | 作用 |
|---|---|
| `llm_factories` | 模型厂商目录 |
| `llm` | 厂商下具体模型清单与能力标签 |
| `tenant_llm` | 租户接入自定义模型与配额 |
| `tenant_langfuse` | 租户的 Langfuse 观测配置 |

### 1.3 知识库与文档管理
| 表名 | 作用 |
|---|---|
| `knowledgebase` | 知识库元数据与默认解析/检索参数 |
| `document` | 文档元数据、解析进度与统计 |
| `file` | 文件/文件夹元信息（资源管理） |
| `file2document` | 文件与文档映射关系 |
| `task` | 文档解析任务与进度、重试和产物 |

### 1.4 对话与检索应用
| 表名 | 作用 |
|---|---|
| `dialog` | 对话应用配置（检索+生成） |
| `conversation` | 某对话应用下的会话与消息摘要 |
| `search` | 独立搜索配置（跨 KB/Doc、检索/重排/高亮等） |

### 1.5 API与集成管理
| 表名 | 作用 |
|---|---|
| `api_token` | API 访问令牌（可绑定应用/来源） |
| `api_4_conversation` | API 调用产生的会话与用量/错误日志 |
| `mcp_server` | MCP Server 注册与调用配置 |

### 1.6 工作流与编排
| 表名 | 作用 |
|---|---|
| `user_canvas` | 用户画布（编排/工作流）实例 |
| `canvas_template` | 画布模板库 |
| `user_canvas_version` | 画布版本历史 |

### 1.7 核心业务流程
- **用户管理** → **模型配置** → **知识库构建** → **应用创建** → **API调用**
- **文档上传** → **任务解析** → **向量化存储** → **检索对话** → **工作流编排**

## 2. 知识库与文档管理
| 表名 | 作用 |
|---|---|
| `knowledgebase` | 知识库元数据与默认解析/检索参数 |
| `document` | 文档元数据、解析进度与统计 |
| `file` | 文件/文件夹元信息（资源管理） |
| `file2document` | 文件与文档映射关系 |
| `task` | 文档解析任务与进度、重试和产物 |

### 2.1 层级关系
知识库与文档管理有三个表: `knowledgebase`、`document`、`file`。它们其实是三层组织关系，从大到小逐级包含：

---

#### **KnowledgeBase (知识库)**

* **最顶层的逻辑容器**，代表一个完整的知识集合。
* 每个 KnowledgeBase 对应一个主题或应用场景（例如“法律法规库”“产品手册库”）。
* **里面包含多个 Document**，是对知识的逻辑分组。
* 在 RAG 检索时，用户通常是指定一个或多个知识库进行检索。

---

#### **Document (文档)**

* **知识库中的一个条目**，逻辑上表示一份“文档”。
* 一个 Document 可能是用户上传的一份 PDF/Word，也可能是一个网页爬取的结果。
* **Document 与 File 是一对多关系**：

  * 一个 Document 可以由多个 File 组成（例如一本书的多章、一个长 PDF 被拆分成多个文件存储）。
* 在 RAG 处理时，Document 会被解析、分片（chunk），然后存入向量数据库中。
* 在 UI 上你看到的“某篇文档”，对应的就是 Document。

---

#### **File (文件)**


* **最底层的物理文件实体**，即用户上传的文件，或者 RAGFlow 爬取/同步下来的原始文件。
* File 是 **原始输入数据**，Document 是它的逻辑表示。
* 文件可能是：

  * 本地上传的 PDF、TXT、DOCX、Markdown
  * 爬取下来的 HTML
  * 甚至是数据库导出的 CSV
* 系统会对 File 做解析（parser → chunk → embedding），再挂载到对应的 Document。

### 2.2 KnowledgeBase

| 字段名                            | 类型                    | 含义                                                  |
| ------------------------------ | --------------------- | --------------------------------------------------- |
| **id**                         | CharField(32), PK     | 知识库 ID，主键，通常是 UUID                                  |
| **avatar**                     | TextField             | 知识库的头像（base64 编码存储图片）                               |
| **tenant_id**                 | CharField(32), index  | 租户 ID，多租户场景下用来区分不同用户/组织的数据                          |
| **name**                       | CharField(128), index | 知识库名称                                               |
| **language**                   | CharField(32), index  | 知识库语言，默认根据系统语言环境设置（"English" 或 "Chinese"）           |
| **description**                | TextField             | 知识库描述                                               |
| **embd_id**                   | CharField(128), index | 默认的向量化模型 ID（embedding model ID），决定文档切分后用哪个模型转向量     |
| **permission**                 | CharField(16), index  | 权限范围，`me`（个人可见）或 `team`（团队可见）                       |
| **created_by**                | CharField(32), index  | 创建者 ID                                              |
| **doc_num**                   | IntegerField, index   | 知识库下的文档数量                                           |
| **token_num**                 | IntegerField, index   | 知识库下的 token 总数（用于控制用量或计费）                           |
| **chunk_num**                 | IntegerField, index   | 知识库下的切片（chunk）数量                                    |
| **similarity_threshold**      | FloatField, index     | 语义检索时的相似度阈值（低于该阈值的结果不会返回），默认 0.2                    |
| **vector_similarity_weight** | FloatField, index     | 召回时向量相似度的权重，用于与其他权重（如关键词检索）混合计算，默认 0.3              |
| **parser_id**                 | CharField(32), index  | 默认文档解析器 ID，决定上传文件时用什么 parser（如 naive、pdf_parser 等） |
| **parser_config**             | JSONField             | 解析器配置，默认 `{ "pages": [[1, 1000000]] }`，表示解析所有页      |
| **pagerank**                   | IntegerField          | 知识库排序权重（比如在 UI 中排序展示时使用）                            |
| **status**                     | CharField(1), index   | 知识库状态，`0=无效`，`1=有效`，默认 1                            |

---

✅ 简单总结：
`knowledgebase` 表是 RagFlow 的 **顶层业务实体**，
主要存储 **知识库的基本信息（id、name、desc、avatar、tenant_id）**，
**运行配置（embd_id、parser_id、similarity_threshold）**，
以及 **统计信息（doc_num、token_num、chunk_num）**。

### 2.3 Document

| 字段名                    | 类型                    | 含义                                                            |
| ---------------------- | --------------------- | ------------------------------------------------------------- |
| **id**                 | CharField(32), PK     | 文档 ID（主键，通常是 UUID）                                            |
| **thumbnail**          | TextField             | 文档的缩略图（base64 存储的图片），用于前端预览                                   |
| **kb_id**             | CharField(256), index | 所属知识库 ID（对应 `knowledgebase.id`，外键关系）                          |
| **parser_id**         | CharField(32), index  | 文档解析器 ID（决定如何解析文档，如 pdf_parser、docx_parser 等）               |
| **parser_config**     | JSONField             | 解析器配置，默认 `{ "pages": [[1, 1000000]] }`，表示解析全部页                |
| **source_type**       | CharField(128), index | 文档来源，默认 `"local"`，也可以是 `"url"`, `"s3"` 等                      |
| **type**               | CharField(32), index  | 文件类型（扩展名，如 `pdf`、`docx`）                                      |
| **created_by**        | CharField(32), index  | 上传文档的用户 ID                                                    |
| **name**               | CharField(255), index | 文件名称                                                          |
| **location**           | CharField(255), index | 文件存储路径（可能是本地路径或远程存储地址）                                        |
| **size**               | IntegerField, index   | 文件大小（字节数）                                                     |
| **token_num**         | IntegerField, index   | 文档解析后切分出的 Token 数量                                            |
| **chunk_num**         | IntegerField, index   | 文档切分后的 chunk 数量                                               |
| **progress**           | FloatField, index     | 文档解析进度（0 \~ 1 之间）                                             |
| **progress_msg**      | TextField             | 解析过程中的状态信息（日志/错误信息等），默认空字符串                                   |
| **process_begin_at** | DateTimeField, index  | 文档解析开始时间                                                      |
| **process_duration**  | FloatField            | 文档解析耗时（秒）                                                     |
| **meta_fields**       | JSONField             | 自定义元信息，默认 `{}`，可存放额外字段                                        |
| **suffix**             | CharField(32), index  | 文件的真实后缀名（例如 `.tar.gz` 的情况，`type` 可能是 `gz`，而 `suffix` 会记录完整信息） |
| **run**                | CharField(1), index   | 文档是否正在处理：`1=运行`，`2=取消`，`0=默认`                                 |
| **status**             | CharField(1), index   | 文档是否有效：`0=无效`，`1=有效`，默认 `1`                                   |

---

✅ 总结：

* `document` 表是 RagFlow 中 **知识库下的具体文档记录**。
* 它存储了文档的 **基本信息（id、、nametype、size、location）**，
  **解析配置（parser_id、parser_config）**，
  **统计指标（token_num、chunk_num、progress）**，
  **状态控制（run、status）**。
* 它与 `knowledgebase` 表通过 `kb_id` 关联，和 **文件存储表(file)** 有下游关系。


### 2.4 File

| 字段名              | 类型                    | 含义                                        |
| ---------------- | --------------------- | ----------------------------------------- |
| **id**           | CharField(32), PK     | 文件或文件夹的唯一 ID（通常是 UUID）                    |
| **parent_id**   | CharField(32), index  | 父目录 ID（如果是根目录则可能为特定值，如 `"0"`），用于组织文件树结构   |
| **tenant_id**   | CharField(32), index  | 租户 ID（区分多租户环境下的数据隔离）                      |
| **created_by**  | CharField(32), index  | 文件或文件夹的创建者（用户 ID）                         |
| **name**         | CharField(255), index | 文件或文件夹名称                                  |
| **location**     | CharField(255), index | 文件存储路径（物理路径或远程存储地址）                       |
| **size**         | IntegerField, index   | 文件大小（单位字节），文件夹则可能为 0                      |
| **type**         | CharField(32), index  | 文件扩展名（例如 `pdf`, `docx`，文件夹可能特殊标记）         |
| **source_type** | CharField(128), index | 文件来源（如 `"local"`、`"s3"`、`"url"` 等），默认空字符串 |

---

✅ 总结

* `file` 表用来管理 **文件系统层面的资源**，既可以表示文件，也可以表示目录（通过 `parent_id` 形成树形结构）。
* 与 `document` 表的区别：

  * `file` 是 **存储层**，关心的是目录/文件的层次、来源、位置。
  * `document` 是 **知识库解析层**，关心的是文件的内容解析、切分、token 统计等。
* 在业务流程中：

  1. 用户上传文件 → 存储在 `file` 表中（带路径、大小、来源）。
  2. 选择文件并导入知识库 → 在 `document` 表中生成一条记录，用于解析处理。
  3. 文档解析 → 生成 `chunk`（切分文本）并写入向量库。


### 2.5 Task

| 字段名                   | 类型                   | 含义                                                   |
| --------------------- | -------------------- | ---------------------------------------------------- |
| **id**                | CharField(32), PK    | 任务唯一 ID（一般是 UUID）                                    |
| **doc_id**           | CharField(32), index | 关联的文档 ID（对应 `document.id`），表示该任务属于哪个文档               |
| **from_page**        | IntegerField         | 起始页码（默认 0），任务处理文档的开始页                                |
| **to_page**          | IntegerField         | 结束页码（默认 100000000，大页码表示“到最后一页”）                      |
| **task_type**        | CharField(32)        | 任务类型（如 `"parse"` 文档解析、 `"chunk"` 切分、 `"embed"` 向量化等） |
| **priority**          | IntegerField         | 任务优先级，数值越高优先执行（通常用于调度队列）                             |
| **begin_at**         | DateTimeField        | 任务开始执行的时间                                            |
| **process_duration** | FloatField           | 任务耗时（秒）                                              |
| **progress**          | FloatField, index    | 任务进度（0\~1，或百分比），比如 0.5 表示 50%                        |
| **progress_msg**     | TextField            | 进度说明信息（例如 `"正在切分第 10 页"`）                            |
| **retry_count**      | IntegerField         | 任务重试次数（失败后会重试）                                       |
| **digest**            | TextField            | 任务摘要/说明（如任务的配置快照，用于对比）                               |
| **chunk_ids**        | LongTextField        | 任务处理生成的 chunk ID 列表（例如解析结果对应的文本块 ID）                 |

✅ 总结

* `task` 表是 RagFlow **异步任务调度系统**的核心表，用来跟踪每个文档处理步骤。
* 一般流程：

  1. 用户上传文档 → 生成 `document` 记录。
  2. 系统为文档生成解析任务 → 在 `task` 表插入一条记录。
  3. Worker 消费任务，更新 `progress`、`progress_msg`。
  4. 任务完成 → 关联的 `chunk_ids` 存储到向量库，文档状态更新。
* `retry_count` 用于容错，`priority` 用于调度，`digest` 用于记录任务执行时的参数快照。


## 3. 模型与AI能力管理
| 表名 | 主要字段/关联 | 作用说明 |
|---|---|---|
| `llm_factories` | 模型厂商目录 |
| `llm` | 厂商下具体模型清单与能力标签 |
| `tenant_llm` | 租户接入模型与配额 |
| `tenant_langfuse` | 租户绑定的具体模型配置（包含 API Key、Base URL、额度等） |


### 3.1 模型厂商目录

| 字段名        | 类型                                                | 含义                                                                                                                           |
| ---------- | ------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| **name**   | CharField(max_length=128, primary_key=True)     | LLM 工厂（模型提供方）的名称，作为主键，例如 `OpenAI`、`Azure`、`Anthropic`、`ZhipuAI`                                                              |
| **logo**   | TextField(null=True)                              | 该工厂的 Logo（Base64 编码字符串），用于前端展示                                                                                               |
| **tags**   | CharField(max_length=255, null=False)            | 工厂支持的能力标签，典型值：<br> - `LLM`（大语言模型）<br> - `Text Embedding`（文本向量化模型）<br> - `Image2Text`（图像转文字，例如 OCR/Caption）<br> - `ASR`（语音识别） |
| **status** | CharField(max_length=1, default="1", index=True) | 工厂状态：<br> - `0` = 废弃/不可用<br> - `1` = 可用（有效）                                                                                  |


### 3.2 模型清单

| 字段名             | 类型                                                | 含义                                                                                                       |
| --------------- | ------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| **llm_name**   | CharField(max_length=128, index=True)            | 模型名称，例如：`gpt-4`、`text-embedding-ada-002`、`claude-3`、`glm-4`                                              |
| **model_type** | CharField(max_length=128, index=True)            | 模型类型，典型值：<br> - `LLM`（大语言模型）<br> - `Text Embedding`（文本向量化模型）<br> - `Image2Text`（图像转文字）<br> - `ASR`（语音识别） |
| **fid**         | CharField(max_length=128, index=True)            | 模型所属工厂 ID（即 **LLMFactories.name**），例如：`OpenAI`、`Anthropic`、`ZhipuAI`                                     |
| **max_tokens** | IntegerField(default=0)                           | 模型支持的最大 token 长度，例如 `4096`、`8192`、`32768`                                                                |
| **tags**        | CharField(max_length=255, index=True)            | 模型标签，进一步描述模型特性，如：<br> - `LLM`（通用大模型）<br> - `Text Embedding`（向量化）<br> - `Chat`（对话式模型）<br> - `32k`（长上下文模型） |
| **is_tools**   | BooleanField(default=False)                       | 是否支持 **工具调用（function calling / tool use）**，例如 GPT-4 Function Calling                                     |
| **status**      | CharField(max_length=1, default="1", index=True) | 模型状态：<br> - `0` = 废弃/不可用<br> - `1` = 有效/可用                                                               |


### 3.3 租户模型配置
租户绑定的具体模型配置（包含 API Key、Base URL、额度等），是**租户层级的实际使用入口**

| 字段名              | 类型                                                 | 含义                                                                                                    |
| ---------------- | -------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| **tenant_id**   | CharField(max_length=32, index=True)              | 租户 ID，表示该配置属于哪个租户（多租户场景下区分用户/团队）                                                                      |
| **llm_factory** | CharField(max_length=128, index=True)             | LLM 工厂名称（对应 **LLMFactories.name**），如：`OpenAI`、`Anthropic`、`ZhipuAI`                                   |
| **model_type**  | CharField(max_length=128, index=True)             | 模型类型，例如：<br> - `LLM`（大语言模型）<br> - `Text Embedding`（文本向量化）<br> - `Image2Text`（图像转文字）<br> - `ASR`（语音识别） |
| **llm_name**    | CharField(max_length=128, index=True, default="") | 模型名称（对应 **LLM.llm_name**），如：`gpt-4`、`text-embedding-ada-002`                                         |
| **api_key**     | CharField(max_length=2048, index=True)            | 租户配置的 **模型 API Key**（通常由租户自己提供）                                                                       |
| **api_base**    | CharField(max_length=255)                         | 模型服务的 **API Base 地址**（例如：`https://api.openai.com/v1`，或私有化部署的 endpoint）                                |
| **max_tokens**  | IntegerField(default=8192, index=True)             | 租户设置的该模型最大可用 token 数（默认 8192，可覆盖模型本身的默认值）                                                             |
| **used_tokens** | IntegerField(default=0, index=True)                | 已使用的 token 数，用于计费/限额统计                                                                                |



### 3.4 租户 Langfuse 配置
* 这张表主要是 **绑定租户和 Langfuse 的监控配置**，让不同租户能将调用链路、日志、评测数据上报到自己专属的 Langfuse 实例。


| 字段名             | 类型                                           | 含义                                    |
| --------------- | -------------------------------------------- | ------------------------------------- |
| **tenant_id**  | CharField(max_length=32, primary_key=True) | 租户 ID（多租户系统下的唯一标识），作为主键。              |
| **secret_key** | CharField(max_length=2048, index=True)      | Langfuse 的 **私钥**，用于服务端安全通信（鉴权）。      |
| **public_key** | CharField(max_length=2048, index=True)      | Langfuse 的 **公钥**，通常用于客户端 SDK 上报。     |
| **host**        | CharField(max_length=128, index=True)       | Langfuse 服务的 **接入地址**（Host/Endpoint）。 |


## 4. 对话与检索应用
| 表名 | 作用 |
|---|---|
| `dialog` | 对话应用配置（检索+生成） |
| `conversation` | 某对话应用下的会话与消息摘要 |
| `search` | 独立搜索配置（跨 KB/Doc、检索/重排/高亮等） |

### 4.1 dialog

| 字段名                            | 类型               | 含义                                                                                                     |
| ------------------------------ | ---------------- | ------------------------------------------------------------------------------------------------------ |
| **id**                         | `CharField(32)`  | 主键，唯一标识一个对话应用                                                                                          |
| **tenant_id**                 | `CharField(32)`  | 租户 ID（多租户隔离用），索引字段                                                                                     |
| **name**                       | `CharField(255)` | 对话应用名称，带索引                                                                                             |
| **description**                | `TextField`      | 对话应用的描述信息                                                                                              |
| **icon**                       | `TextField`      | 应用图标的 base64 编码字符串                                                                                     |
| **language**                   | `CharField(32)`  | 语言（English / Chinese），默认根据系统环境变量 `LANG` 设置，带索引                                                         |
| **llm_id**                    | `CharField(128)` | 默认的大语言模型（LLM）ID                                                                                        |
| **llm_setting**               | `JSONField`      | LLM 配置参数（默认：`temperature=0.1, top_p=0.3, frequency_penalty=0.7, presence_penalty=0.4, max_tokens=512`） |
| **prompt_type**               | `CharField(16)`  | Prompt 模式，`simple` 或 `advanced`，默认 `simple`，带索引                                                        |
| **prompt_config**             | `JSONField`      | Prompt 配置，包含 system、prologue（开场白）、parameters（参数列表）、empty_response（知识库无匹配时的回复）                         |
| **meta_data_filter**         | `JSONField`      | 元数据过滤条件（例如按标签/属性筛选知识库内容），默认 `{}`                                                                       |
| **similarity_threshold**      | `FloatField`     | 向量召回的相似度阈值（默认 `0.2`）                                                                                   |
| **vector_similarity_weight** | `FloatField`     | 向量相似度的权重（默认 `0.3`）                                                                                     |
| **top_n**                     | `IntegerField`   | 向量检索时返回的候选文档数量（默认 `6`）                                                                                 |
| **top_k**                     | `IntegerField`   | 候选文档池的最大数量（默认 `1024`）                                                                                  |
| **do_refer**                  | `CharField(1)`   | 是否在答案中插入引用信息（默认 `"1"` 表示需要）                                                                            |
| **rerank_id**                 | `CharField(128)` | 默认的 rerank（重排序）模型 ID                                                                                   |
| **kb_ids**                    | `JSONField`      | 绑定的知识库 ID 列表（默认空数组 `[]`）                                                                               |
| **status**                     | `CharField(1)`   | 应用状态（`0`: 废弃 / 无效，`1`: 有效），默认 `"1"`，带索引                                                                |


### 4.2 conversation

| 字段名            | 类型               | 含义                             |
| -------------- | ---------------- | ------------------------------ |
| **id**         | `CharField(32)`  | 主键，唯一标识一个会话                    |
| **dialog_id** | `CharField(32)`  | 关联的对话应用（`dialog` 表的 `id`），索引字段 |
| **name**       | `CharField(255)` | 会话名称（可自定义），索引字段                |
| **message**    | `JSONField`      | 会话消息内容（通常存储消息历史、对话上下文等）        |
| **reference**  | `JSONField`      | 会话相关的参考信息（如知识库引用），默认空数组 `[]`   |
| **user_id**   | `CharField(255)` | 发起会话的用户 ID，索引字段                |

---

📌 总结：

* `Dialog` 是 **应用配置**，决定了这个对话应用如何运行（用哪个 LLM、知识库、检索参数等）。
* `Conversation` 是 **实际的会话实例**，保存具体的对话记录（message、reference）、用户归属（user_id）、以及属于哪个应用（dialog_id）。


### 4.3 search

| 字段名                              | 类型               | 含义                                    |
| -------------------------------- | ---------------- | ------------------------------------- |
| **id**                           | `CharField(32)`  | 主键，唯一标识一个搜索配置                         |
| **avatar**                       | `TextField`      | 搜索配置的头像（base64 编码字符串）                 |
| **tenant_id**                   | `CharField(32)`  | 所属租户 ID，索引字段                          |
| **name**                         | `CharField(128)` | 搜索配置名称，必填，索引字段                        |
| **description**                  | `TextField`      | 搜索配置描述（通常是对应知识库的说明）                   |
| **created_by**                  | `CharField(32)`  | 创建者的用户 ID，索引字段                        |
| **search_config**               | `JSONField`      | 搜索配置（核心字段），包含以下子项：                    |
| └─ `kb_ids`                      | `list`           | 知识库 ID 列表                             |
| └─ `doc_ids`                     | `list`           | 文档 ID 列表                              |
| └─ `similarity_threshold`        | `float`          | 相似度阈值，默认 0.2                          |
| └─ `vector_similarity_weight`    | `float`          | 向量相似度权重，默认 0.3                        |
| └─ `use_kg`                      | `bool`           | 是否使用知识图谱（Knowledge Graph）             |
| └─ `rerank_id`                   | `str`            | 重新排序模型 ID                             |
| └─ `top_k`                       | `int`            | 最大召回候选数量，默认 1024                      |
| └─ `summary`                     | `bool`           | 是否启用结果摘要                              |
| └─ `chat_id`                     | `str`            | 关联的会话 ID                              |
| └─ `llm_setting`                 | `dict`           | LLM 配置（temperature、top_p 等，可选，不设默认值） |
| └─ `chat_settingcross_languages` | `list`           | 跨语言聊天设置                               |
| └─ `highlight`                   | `bool`           | 是否高亮匹配片段                              |
| └─ `keyword`                     | `bool`           | 是否启用关键词检索                             |
| └─ `web_search`                  | `bool`           | 是否启用联网搜索                              |
| └─ `related_search`              | `bool`           | 是否启用相关搜索推荐                            |
| └─ `query_mindmap`               | `bool`           | 是否启用查询思维导图                            |
| **status**                       | `CharField(1)`   | 配置状态，`1` 表示有效，`0` 表示无效，索引字段           |

---

📌 总结：

* `Search` 表就是一个 **搜索配置实体**，核心在 `search_config`，里面把知识库、文档、检索参数、是否用 LLM、是否联网搜索等全部配好。
* 这样每个 `Search` 实例就代表一种 **定制化搜索方案**，可以由不同租户/用户创建和使用。


## 5. 工作流与编排
| 表名 | 作用 |
|---|---|
| `user_canvas` | 用户画布（编排/工作流）实例 |
| `canvas_template` | 画布模板库 |
| `user_canvas_version` | 画布版本历史 |

### 5.1 user_canvas

| 字段名           | 类型        | 含义                               |
| ------------- | --------- | -------------------------------- |
| `id`          | CharField | Canvas 的唯一标识，主键                  |
| `avatar`      | TextField | Canvas 头像，存储为 Base64 字符串         |
| `user_id`     | CharField | 用户 ID，标识 Canvas 所属用户             |
| `title`       | CharField | Canvas 标题                        |
| `permission`  | CharField | 权限类型，可选 `"me"`（私有）或 `"team"`（团队） |
| `description` | TextField | Canvas 描述                        |
| `canvas_type` | CharField | Canvas 类型                        |
| `dsl`         | JSONField | Canvas 的 DSL 数据，存储 JSON          |


### 5.2 canvas_template

| 字段名           | 类型        | 含义                          |
| ------------- | --------- | --------------------------- |
| `id`          | CharField | Canvas 模板的唯一标识，主键           |
| `avatar`      | TextField | Canvas 模板的头像，存储为 Base64 字符串 |
| `title`       | CharField | Canvas 模板标题                 |
| `description` | TextField | Canvas 模板描述                 |
| `canvas_type` | CharField | Canvas 类型                   |
| `dsl`         | JSONField | Canvas 模板的 DSL 数据，存储 JSON   |

### 5.3 user_canvas_version

| 字段名              | 类型        | 含义                              |
| ---------------- | --------- | ------------------------------- |
| `id`             | CharField | Canvas 版本的唯一标识，主键               |
| `user_canvas_id` | CharField | 所属的 UserCanvas ID，用于关联具体 Canvas |
| `title`          | CharField | Canvas 版本标题                     |
| `description`    | TextField | Canvas 版本描述                     |
| `dsl`            | JSONField | Canvas 版本的 DSL 数据，存储 JSON       |


## 6. API与集成管理
| 表名 | 作用 |
|---|---|
| `user` | 用户账号信息与登录属性 |
| `tenant` | 租户/工作区，默认模型与额度配置 |
| `user_tenant` | 用户与租户的关系与角色 |
| `invitation_code` | 邀请码与受邀记录 |
| `api_token` | API 访问令牌（可绑定应用/来源） |
| `api_4_conversation` | API 调用产生的会话与用量/错误日志 |
| `mcp_server` | MCP Server 注册与调用配置 |

### 6.1 user

| 字段名                   | 类型               | 含义                                                  |                        |
| --------------------- | ---------------- | --------------------------------------------------- | ---------------------- |
| **id**                | `CharField(32)`  | 主键，唯一标识用户                                           |                        |
| **access_token**     | `CharField(255)` | 用户的访问令牌（用于鉴权），索引字段                                  |                        |
| **nickname**          | `CharField(100)` | 用户昵称，必填，索引字段                                        |                        |
| **password**          | `CharField(255)` | 登录密码（加密存储），索引字段                                     |                        |
| **email**             | `CharField(255)` | 邮箱，必填，用户的主要登录标识，索引字段                                |                        |
| **avatar**            | `TextField`      | 用户头像（base64 编码字符串）                                  |                        |
| **language**          | `CharField(32)`  | 用户语言偏好，默认根据系统 `LANG` 变量设置（"Chinese"/"English"），索引字段 |                        |
| **color_schema**     | `CharField(32)`  | 颜色主题（Bright                                         | Dark），默认 "Bright"，索引字段 |
| **timezone**          | `CharField(64)`  | 用户时区，默认 `"UTC+8 Asia/Shanghai"`，索引字段                |                        |
| **last_login_time** | `DateTimeField`  | 最近一次登录时间，索引字段                                       |                        |
| **is_authenticated** | `CharField(1)`   | 是否已认证，默认 `"1"`，索引字段                                 |                        |
| **is_active**        | `CharField(1)`   | 是否活跃用户，默认 `"1"`，索引字段                                |                        |
| **is_anonymous**     | `CharField(1)`   | 是否匿名用户，默认 `"0"`，索引字段                                |                        |
| **login_channel**    | `CharField`      | 用户登录来源（如 web、第三方登录），索引字段                            |                        |
| **status**            | `CharField(1)`   | 用户状态，`1` 有效，`0` 无效，默认 `"1"`，索引字段                    |                        |
| **is_superuser**     | `BooleanField`   | 是否超级管理员（root 权限），默认 `False`，索引字段                    |                        |

---

额外说明

* **继承关系**

  * `DataBaseModel`：应用的 Peewee 基础模型，封装了通用数据库配置。
  * `UserMixin`：通常来自 `flask_login`，提供用户认证相关的默认方法（如 `is_authenticated`、`is_active` 等）。

* **方法**

  * `__str__`：返回用户邮箱，便于打印或调试。
  * `get_id`：使用 `itsdangerous.Serializer` 对 `access_token` 进行加密后返回，通常用于会话或 JWT 登录鉴权。


### 6.2 tenant

| 字段名             | 类型               | 含义                                     |
| --------------- | ---------------- | -------------------------------------- |
| **id**          | `CharField(32)`  | 主键，租户唯一标识                              |
| **name**        | `CharField(100)` | 租户名称，索引字段                              |
| **public_key** | `CharField(255)` | 租户公钥，用于鉴权或 API 调用，索引字段                 |
| **llm_id**     | `CharField(128)` | 默认使用的大语言模型（LLM）的 ID，索引字段               |
| **embd_id**    | `CharField(128)` | 默认使用的向量化（embedding）模型 ID，索引字段          |
| **asr_id**     | `CharField(128)` | 默认使用的语音识别（ASR）模型 ID，索引字段               |
| **img2txt_id** | `CharField(128)` | 默认使用的图像转文本模型 ID，索引字段                   |
| **rerank_id**  | `CharField(128)` | 默认使用的重排序模型 ID，索引字段                     |
| **tts_id**     | `CharField(256)` | 默认使用的文本转语音（TTS）模型 ID，索引字段              |
| **parser_ids** | `CharField(256)` | 文档解析器（document processors）的 ID 列表，索引字段 |
| **credit**      | `IntegerField`   | 租户可用额度（如调用次数/积分），默认 `512`，索引字段         |
| **status**      | `CharField(1)`   | 租户状态：`1` 有效，`0` 无效，默认 `"1"`，索引字段       |

---

额外说明

* **租户概念**

  * 一般 SaaS 系统中的 **多租户隔离设计**。
  * 每个租户可以配置自己的 **默认模型**（LLM、Embedding、ASR、TTS 等）。
  * `credit` 可理解为额度或余额控制。

* **模型关联**

  * `llm_id`、`embd_id`、`asr_id`、`tts_id` 等字段通常会对应到 **模型表**（或外部模型服务 ID），便于快速切换。
  * `parser_ids` 说明每个租户可以绑定一组 **文档预处理器**（例如 OCR、分词、结构化提取等）。

---

📌 总结：
`Tenant` 表定义了 **多租户体系的配置中心**，主要作用是：

1. 标识不同租户（公司/组织）。
2. 存储每个租户的默认模型配置。
3. 控制租户的可用额度和有效状态。


### 6.3 user_tenant

| 字段名             | 类型              | 含义                                     |
| --------------- | --------------- | -------------------------------------- |
| **id**          | `CharField(32)` | 主键，用户-租户关系唯一标识                         |
| **user_id**    | `CharField(32)` | 用户 ID，关联到 `user.id`，索引字段               |
| **tenant_id**  | `CharField(32)` | 租户 ID，关联到 `tenant.id`，索引字段             |
| **role**        | `CharField(32)` | 用户在该租户下的角色（例如 `admin`、`member` 等），索引字段 |
| **invited_by** | `CharField(32)` | 邀请该用户加入租户的 **邀请人 ID**（通常也是某个用户），索引字段   |
| **status**      | `CharField(1)`  | 用户-租户关系的状态：`1` 有效，`0` 无效，默认 `"1"`，索引字段 |

---

额外说明

* **角色管理（role 字段）**

  * 用来区分用户在租户中的权限，例如：

    * `owner`：租户创建者
    * `admin`：租户管理员
    * `member`：普通成员
  * 在 RBAC（基于角色的权限控制）里非常常见。

### 6.4 invitation_code

| 字段名             | 类型              | 含义                                    |
| --------------- | --------------- | ------------------------------------- |
| **id**          | `CharField(32)` | 主键，邀请码记录唯一标识                          |
| **code**        | `CharField(32)` | 邀请码字符串，作为唯一识别和校验的凭证，索引字段              |
| **visit_time** | `DateTimeField` | 邀请码被使用或访问的时间，用于追踪使用情况                 |
| **user_id**    | `CharField(32)` | 使用该邀请码的用户 ID，索引字段                     |
| **tenant_id**  | `CharField(32)` | 该邀请码所属的租户 ID，索引字段                     |
| **status**      | `CharField(1)`  | 邀请码状态：`1` 有效，`0` 失效（废弃），默认 `"1"`，索引字段 |


### 6.5 api_token

| 字段名            | 类型               | 含义                                                                                              |
| -------------- | ---------------- | ----------------------------------------------------------------------------------------------- |
| **tenant_id** | `CharField(32)`  | 租户 ID，表明该 API Token 属于哪个租户，主键的一部分                                                               |
| **token**      | `CharField(255)` | API Token 字符串（唯一识别 API 调用者的凭证），主键的一部分                                                           |
| **dialog_id** | `CharField(32)`  | 关联的对话应用 ID（如果该 Token 限定在某个对话应用场景下使用），可为空                                                        |
| **source**     | `CharField(16)`  | Token 来源，用于标记用途：<br> - `none`：普通用途 <br> - `agent`：代理模式下生成的 Token <br> - `dialog`：对话应用下生成的 Token |
| **beta**       | `CharField(255)` | 预留字段（实验功能标识或 Beta 功能开关），可为空                                                                     |


### 6.6 api_4_conversation
API 调用产生的会话与用量/错误日志

| 字段名            | 类型               | 含义                             |       |               |
| -------------- | ---------------- | ------------------------------ | ----- | ------------- |
| **id**         | `CharField(32)`  | 主键，唯一标识一条 API 会话记录             |       |               |
| **dialog_id** | `CharField(32)`  | 关联的对话应用 ID，索引字段                |       |               |
| **user_id**   | `CharField(255)` | 发起会话的用户 ID，索引字段                |       |               |
| **message**    | `JSONField`      | API 会话中的消息内容（输入/输出历史）          |       |               |
| **reference**  | `JSONField`      | 会话参考信息（如知识库引用），默认空数组 `[]`      |       |               |
| **tokens**     | `IntegerField`   | 会话消耗的 token 数量，默认 `0`          |       |               |
| **source**     | `CharField(16)`  | 会话来源，\`none                    | agent | dialog\`，索引字段 |
| **dsl**        | `JSONField`      | 额外的 DSL 或自定义指令信息，默认空字典 `{}`    |       |               |
| **duration**   | `FloatField`     | 本次会话耗时（秒），默认 `0`，索引字段          |       |               |
| **round**      | `IntegerField`   | 会话轮次（消息轮数），默认 `0`，索引字段         |       |               |
| **thumb_up**  | `IntegerField`   | 用户点赞数（对本轮或本次会话的评价），默认 `0`，索引字段 |       |               |
| **errors**     | `TextField`      | 会话处理中的错误信息，便于调试或日志追踪           |       |               |

---

额外说明

* **用途**

  * 用于记录 **通过 API 发起的会话**，区别于前端直接的 `Conversation`。
  * 支持统计 token 使用量、响应耗时和用户评价。

* **关联关系**

  * `dialog_id` → 对应某个对话应用（Dialog）。
  * `user_id` → 记录是哪位用户发起的 API 调用。
  * `source` → 区分调用来源：

    * `none`：默认
    * `agent`：代理调用
    * `dialog`：通过对话应用触发

* **性能/统计指标**

  * `tokens`、`duration`、`round`、`thumb_up` 可以用于 **计费、性能监控、用户行为分析**。

---

📌 总结：
`API4Conversation` 是 **API 层会话日志表**，比 `Conversation` 更偏向 **调用监控和统计**，记录 token 消耗、耗时、轮次、来源和错误信息，方便 **审计与分析 API 使用情况**。


### 6.7 mcp_server
👌 我来帮你整理 `MCPServer` 这张表的字段说明：

---

### `mcp_server` 表字段说明

| 字段名              | 类型                | 含义                               |
| ---------------- | ----------------- | -------------------------------- |
| **id**           | `CharField(32)`   | 主键，唯一标识一个 MCP Server 实例          |
| **name**         | `CharField(255)`  | MCP Server 名称，必填                 |
| **tenant_id**   | `CharField(32)`   | 所属租户 ID，索引字段                     |
| **url**          | `CharField(2048)` | MCP Server 的访问 URL，必填            |
| **server_type** | `CharField(32)`   | MCP Server 类型（例如任务调度、数据处理等），必填   |
| **description**  | `TextField`       | MCP Server 描述信息                  |
| **variables**    | `JSONField`       | MCP Server 配置变量（键值对），默认空字典 `{}`  |
| **headers**      | `JSONField`       | MCP Server 请求头信息（键值对），默认空字典 `{}` |



