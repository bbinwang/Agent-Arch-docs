# 第 5 章：核心代码讲解（上）

> 本章对 LlamaIndex 最核心的 5 个模块进行逐文件、逐函数深度走读：LLM 抽象层、Embedding 抽象层、BaseIndex 与 VectorStoreIndex、BaseRetriever 与 Retriever 层次、QueryEngine 层次。

---

## 5.1 LLM 抽象层

### 5.1.1 BaseLLM（`base/llms/base.py`）

**文件路径**: `llama-index-core/llama_index/core/base/llms/base.py`
**总行数**: ~350 行
**继承链**: `BaseComponent(BaseModel) + DispatcherSpanMixin → BaseLLM`

#### 类定义

```python
class BaseLLM(BaseComponent, DispatcherSpanMixin):
    model_config = ConfigDict(arbitrary_types_allowed=True)
    callback_manager: CallbackManager = Field(
        default_factory=lambda: CallbackManager([]), exclude=True
    )
    rate_limiter: Optional[Any] = Field(default=None, exclude=True)
```

**字段说明**:
- `callback_manager`: 回调管理器，用于事件通知（exclude=True 表示不序列化）
- `rate_limiter`: 速率限制器，用于控制 API 调用频率

#### 抽象方法（子类必须实现）

| 方法 | 签名 | 说明 |
|------|------|------|
| `metadata` | `@property @abstractmethod → LLMMetadata` | LLM 元数据（上下文窗口、是否 Chat 模型等） |
| `chat` | `(messages: Sequence[ChatMessage], **kwargs) → ChatResponse` | 同步对话 |
| `complete` | `(prompt: str, formatted=False, **kwargs) → CompletionResponse` | 同步补全 |
| `stream_chat` | `(messages, **kwargs) → ChatResponseGen` | 同步流式对话 |
| `stream_complete` | `(prompt, **kwargs) → CompletionResponseGen` | 同步流式补全 |
| `achat` | `(messages, **kwargs) → ChatResponse` | 异步对话 |
| `acomplete` | `(prompt, **kwargs) → CompletionResponse` | 异步补全 |
| `astream_chat` | `(messages, **kwargs) → ChatResponseAsyncGen` | 异步流式对话 |
| `astream_complete` | `(prompt, **kwargs) → CompletionResponseAsyncGen` | 异步流式补全 |

#### 关键方法详解

**`to_payload()`** (行 58-66):
```python
def to_payload(self) -> Dict[str, Any]:
    return {"class_name": self.class_name(), **self.metadata.model_dump()}
```
用于可观测性——返回非敏感表示，绝不包含 API Key 等凭证。

**`convert_chat_messages()`** (行 68-86):
将 `ChatMessage` 列表转换为 LLM 特定格式。处理 `TextBlock` 内容块，将多块内容拼接为字符串。

#### LLMMetadata 结构

```python
class LLMMetadata(BaseModel):
    context_window: int = 3900
    num_output: int = 256
    is_chat_model: bool = False
    is_function_calling_model: bool = False
    model_name: str = "unknown"
    is_huggingface_model: bool = False
    # ... 更多字段
```

---

### 5.1.2 LLM（`llms/llm.py`）

**文件路径**: `llama-index-core/llama_index/core/llms/llm.py`
**总行数**: 946 行
**继承链**: `BaseLLM → LLM`

#### 类定义与字段

```python
class LLM(BaseLLM):
    system_prompt: Optional[str] = Field(default=None)
    messages_to_prompt: MessagesToPromptCallable = Field(default=None, exclude=True)
    completion_to_prompt: CompletionToPromptCallable = Field(default=None, exclude=True)
    output_parser: Optional[BaseOutputParser] = Field(default=None, exclude=True)
    pydantic_program_mode: PydanticProgramMode = PydanticProgramMode.DEFAULT
    query_wrapper_prompt: Optional[BasePromptTemplate] = Field(default=None)  # deprecated
```

**字段说明**:
- `system_prompt`: 系统提示，自动添加到每条消息前
- `messages_to_prompt`: 将 ChatMessage 列表转为 prompt 字符串的函数
- `completion_to_prompt`: 将 completion 转为 prompt 的函数
- `output_parser`: 输出解析器，用于结构化输出
- `pydantic_program_mode`: Pydantic 程序模式（DEFAULT/WORKAROUND）

#### 核心方法详解

**`predict(prompt, **prompt_args)`** (行 602-645):
```python
@dispatcher.span
def predict(self, prompt: BasePromptTemplate, **prompt_args) -> str:
    dispatcher.event(LLMPredictStartEvent(template=prompt, template_args=prompt_args))
    self._log_template_data(prompt, **prompt_args)

    if self.metadata.is_chat_model:
        messages = self._get_messages(prompt, **prompt_args)
        chat_response = self.chat(messages)
        output = chat_response.message.content or ""
    else:
        formatted_prompt = self._get_prompt(prompt, **prompt_args)
        response = self.complete(formatted_prompt, formatted=True)
        output = response.text

    parsed_output = self._parse_output(output)
    dispatcher.event(LLMPredictEndEvent(output=parsed_output))
    return parsed_output
```

**逻辑流程**:
1. 发出 `LLMPredictStartEvent` 事件
2. 记录模板数据
3. 根据 `is_chat_model` 选择 `chat()` 或 `complete()`
4. 解析输出（如有 output_parser）
5. 发出 `LLMPredictEndEvent` 事件

**`structured_predict(output_cls, prompt, **prompt_args)`** (行 306-371):
```python
@dispatcher.span
def structured_predict(self, output_cls, prompt, llm_kwargs=None, **prompt_args) -> Model:
    dispatcher.event(LLMStructuredPredictStartEvent(...))
    program = get_program_for_llm(output_cls, prompt, self, self.pydantic_program_mode)
    result = program(llm_kwargs=llm_kwargs, **prompt_args)
    assert not isinstance(result, list)
    if not isinstance(result, BaseModel):
        raise TypeError(...)
    dispatcher.event(LLMStructuredPredictEndEvent(output=result))
    return result
```

**设计要点**:
- 使用 `get_program_for_llm()` 工厂方法选择合适的 Program 实现
- `PydanticProgramMode` 控制使用哪种 Pydantic 程序
- 同步/异步/流式三个版本

**`as_structured_llm(output_cls)`**:
将当前 LLM 包装为 `StructuredLLM`，后续所有调用自动返回 `output_cls` 实例。

#### 辅助方法

| 方法 | 功能 |
|------|------|
| `_log_template_data(prompt, **args)` | 记录模板变量到回调事件 |
| `_get_prompt(prompt, **args)` | 格式化 prompt + 注入 output_parser + 扩展 system_prompt |
| `_get_messages(prompt, **args)` | 格式化为 ChatMessage 列表 + 扩展 system_prompt |
| `_parse_output(output)` | 使用 output_parser 解析输出 |
| `_extend_prompt(formatted)` | 添加 system_prompt + query_wrapper_prompt |
| `_extend_messages(messages)` | 在消息列表前添加 system prompt |

#### 设计模式分析

1. **Template Method**: `predict()` 定义骨架，`chat()`/`complete()` 由子类实现
2. **Strategy**: `messages_to_prompt` / `completion_to_prompt` 是可注入策略
3. **Decorator**: `@dispatcher.span` 装饰器自动包装 Instrumentation Span
4. **Factory**: `get_program_for_llm()` 根据 LLM 类型创建合适的 Program

#### 潜在问题与改进

| 问题 | 影响 | 改进建议 |
|------|------|----------|
| `messages_to_prompt` 使用 `Callable` 类型 | 不支持序列化 | 使用 `exclude=True` 规避 |
| `query_wrapper_prompt` 已弃用 | 代码冗余 | 在下一主版本移除 |
| `structured_predict` 的 `assert` 不生产环境友好 | 可能抛出 AssertionError | 使用显式异常 |

---

### 5.1.3 StructuredLLM（`llms/structured_llm.py`）

**文件路径**: `llama-index-core/llama_index/core/llms/structured_llm.py`

`StructuredLLM` 是 LLM 的**装饰器/包装器**，将任意 LLM 转换为结构化输出版本：

```python
class StructuredLLM(BaseLLM):
    llm: BaseLLM
    output_cls: Type[BaseModel]

    def complete(self, prompt, **kwargs):
        output = self.llm.complete(prompt, **kwargs)
        return self._prepare_output(output.text)

    def chat(self, messages, **kwargs):
        output = self.llm.chat(messages, **kwargs)
        return self._prepare_output(output.message.content)
```

**设计模式**: Decorator（装饰器模式）——不改变原 LLM 接口，增加结构化输出能力。

---

## 5.2 Embedding 抽象层

### 5.2.1 BaseEmbedding（`base/embeddings/base.py`）

**文件路径**: `llama-index-core/llama_index/core/base/embeddings/base.py`
**继承链**: `TransformComponent + DispatcherSpanMixin → BaseEmbedding`

#### 抽象方法

| 方法 | 说明 |
|------|------|
| `_get_query_embedding(query) → List[float]` | 查询向量（子类实现） |
| `_aget_query_embedding(query) → List[float]` | 异步查询向量 |
| `_get_text_embedding(text) → List[float]` | 文本向量（子类实现） |
| `_aget_text_embedding(text) → List[float]` | 异步文本向量 |

#### 公共方法

| 方法 | 说明 |
|------|------|
| `get_query_embedding(query)` | 查询向量（带缓存 + 速率限制） |
| `get_text_embedding(text)` | 文本向量 |
| `get_text_embedding_batch(texts, **kwargs)` | 批量文本向量 |
| `aget_text_embedding_batch(texts, **kwargs)` | 异步批量 |
| `__call__(nodes, **kwargs)` | 使 Embedding 可作为 Transform 使用 |

#### 关键实现

**`get_text_embedding_batch()`**:
```python
def get_text_embedding_batch(self, texts, show_progress=False, **kwargs):
    with self.callback_manager.event(CBEventType.EMBEDDING, ...):
        batches = iterate_custom_batch(self._batch_size, texts)
        if self._rate_limiter:
            self._rate_limiter.acquire(len(texts))

        embeddings = []
        for batch in batches:
            if self.embeddings_cache:
                cached = self.embeddings_cache.get(batch)
                # ... 使用缓存
            else:
                embeddings.extend(self._get_text_embedding_batch(batch))
        return embeddings
```

**设计要点**:
- 自动分批处理（`_batch_size`）
- 内置缓存（`embeddings_cache: BaseKVStore`）
- 速率限制（`rate_limiter`）
- 进度条（`show_progress`）

#### SimilarityMode 枚举

```python
class SimilarityMode(str, Enum):
    DEFAULT = "cosine"       # 余弦相似度
    DOT_PRODUCT = "dot_product"  # 内积
    EUCLIDEAN = "euclidean"  # 欧氏距离
```

---

## 5.3 BaseIndex 与 VectorStoreIndex

### 5.3.1 BaseIndex（`indices/base.py`）

**文件路径**: `llama-index-core/llama_index/core/indices/base.py`
**总行数**: 595 行
**继承链**: `Generic[IS] + ABC → BaseIndex`

#### 类定义

```python
class BaseIndex(Generic[IS], ABC):
    index_struct_cls: Type[IS]

    def __init__(
        self,
        nodes=None, objects=None, index_struct=None,
        storage_context=None, callback_manager=None,
        transformations=None, show_progress=False, **kwargs
    ):
        self._storage_context = storage_context or StorageContext.from_defaults()
        self._docstore = self._storage_context.docstore
        self._vector_store = self._storage_context.vector_store
        self._graph_store = self._storage_context.graph_store
        self._callback_manager = callback_manager or Settings.callback_manager
        # ... 构建索引结构
```

#### 三种转换方法（核心设计理念）

```python
def as_retriever(self, **kwargs) -> BaseRetriever: ...
def as_query_engine(self, llm=None, **kwargs) -> BaseQueryEngine: ...
def as_chat_engine(self, chat_mode=ChatMode.BEST, **kwargs) -> BaseChatEngine: ...
```

**设计 Rationale**:
- 同一个 Index 可以转换为三种不同的查询接口
- `as_retriever()` → 仅检索，返回 NodeWithScore 列表
- `as_query_engine()` → 检索 + 合成，返回 Response
- `as_chat_engine()` → 多轮对话，维护 ChatMemory

#### 抽象方法

| 方法 | 说明 |
|------|------|
| `_build_index_from_nodes(nodes) → IS` | 从节点构建索引结构 |
| `_insert(nodes)` | 插入节点 |
| `_delete_node(node_id)` | 删除节点 |
| `as_retriever(**kwargs)` | 创建检索器 |
| `ref_doc_info` | 引用文档信息 |

#### 增量操作

```python
def insert_nodes(self, nodes) -> None: ...
def ainsert_nodes(self, nodes) -> None: ...
def delete_ref_doc(self, ref_doc_id, raise_error=True) -> None: ...
def update_ref_doc(self, ref_doc_id, nodes, **kwargs) -> None: ...
def refresh_ref_docs(self, ref_doc_info) -> bool: ...
```

---

### 5.3.2 VectorStoreIndex（`indices/vector_store/base.py`）

**文件路径**: `llama-index-core/llama_index/core/indices/vector_store/base.py`
**总行数**: 487 行
**继承链**: `BaseIndex[IndexDict] → VectorStoreIndex`

#### 类定义

```python
class VectorStoreIndex(BaseIndex[IndexDict]):
    _insert_batch_size: int = Field(default=2048)
    _use_async: bool = Field(default=True)
    _store_nodes_override: bool = Field(default=False)
    _embed_model: BaseEmbedding = Field(exclude=True)
```

#### 核心方法

**`_build_index_from_nodes(nodes)`** (行 260-284):
```python
def _build_index_from_nodes(self, nodes, **insert_kwargs) -> IndexDict:
    index_struct = self.index_struct_cls()
    if self._use_async:
        tasks = [self._async_add_nodes_to_index(index_struct, nodes, ...)]
        run_async_tasks(tasks)
    else:
        self._add_nodes_to_index(index_struct, nodes, ...)
    return index_struct
```

**`_add_nodes_to_index(index_struct, nodes)`** (行 219-258):
1. 按 `_insert_batch_size` 分批
2. 调用 `_get_node_with_embedding()` 计算嵌入
3. 调用 `vector_store.add()` 写入向量
4. 如果 vector_store 不存储文本，写入 docstore

**`as_retriever(**kwargs)`**:
```python
def as_retriever(self, **kwargs) -> BaseRetriever:
    return VectorIndexRetriever(
        self,
        node_ids=list(self.index_struct.nodes_dict.values()),
        callback_manager=self._callback_manager,
        object_map=self._object_map,
        **kwargs,
    )
```

**`as_query_engine(llm, **kwargs)`**:
```python
def as_query_engine(self, llm=None, **kwargs) -> BaseQueryEngine:
    kwargs["llm"] = llm or Settings.llm
    return RetrieverQueryEngine.from_args(
        retriever=self.as_retriever(**kwargs.pop("retriever_kwargs", {})),
        **kwargs,
    )
```

#### 删除与更新

```python
def delete_ref_doc(self, ref_doc_id, raise_error=True, **kwargs) -> None:
    self._index_struct.delete(ref_doc_id)
    self._vector_store.delete(ref_doc_id, **kwargs)
    self._docstore.delete_document(ref_doc_id)
    self._storage_context.index_store.add_index_struct(self._index_struct)
```

**设计要点**:
- 删除操作需要同步更新 index_struct + vector_store + docstore
- 删除后重新写入 index_store 保持一致性

---

## 5.4 BaseRetriever 与 Retriever 层次

### 5.4.1 BaseRetriever（`base/base_retriever.py`）

**文件路径**: `llama-index-core/llama_index/core/base/base_retriever.py`
**总行数**: 280 行
**继承链**: `PromptMixin + DispatcherSpanMixin → BaseRetriever`

#### 类定义

```python
class BaseRetriever(PromptMixin, DispatcherSpanMixin):
    def __init__(self, callback_manager=None, object_map=None, objects=None, verbose=False):
        self.callback_manager = callback_manager or CallbackManager()
        self.object_map = object_map or {}
        self._verbose = verbose
```

#### 核心方法

**`retrieve(str_or_query_bundle)`**:
```python
def retrieve(self, str_or_query_bundle) -> List[NodeWithScore]:
    if isinstance(str_or_query_bundle, str):
        str_or_query_bundle = QueryBundle(query_str=str_or_query_bundle)

    with self.callback_manager.event(CBEventType.RETRIEVE, ...) as dispatch_event:
        nodes = self._retrieve(str_or_query_bundle)
        dispatch_event.on_end(payload={EventPayload.NODES: nodes})

        # 处理递归检索（IndexNode → object resolution）
        nodes = self._handle_recursive_retrieval(nodes, str_or_query_bundle)

    return nodes
```

**`_handle_recursive_retrieval(nodes, query_bundle)`**:
处理 IndexNode 引用——如果检索到的节点是 IndexNode，则解析其引用的对象（QueryEngine/Retriever/BaseNode）并递归检索。

#### 抽象方法

```python
@abstractmethod
def _retrieve(self, query_bundle: QueryBundle) -> List[NodeWithScore]: ...
```

---

### 5.4.2 VectorIndexRetriever（`indices/vector_store/retrievers/`）

**文件路径**: `llama-index-core/llama_index/core/indices/vector_store/retrievers/retriever.py`

```python
class VectorIndexRetriever(BaseRetriever):
    _index: VectorStoreIndex
    _node_ids: Optional[List[str]]
    _similarity_top_k: int = 1
    _embed_model: BaseEmbedding

    def _retrieve(self, query_bundle):
        if query_bundle.embedding is None:
            query_bundle.embedding = self._embed_model.get_query_embedding(
                query_bundle.query_str
            )

        return self._index.vector_store.query(
            VectorStoreQuery(
                query_embedding=query_bundle.embedding,
                similarity_top_k=self._similarity_top_k,
                node_ids=self._node_ids,
                doc_ids=self._doc_ids,
                query_str=query_bundle.query_str,
                mode=self._mode,
                alpha=self._alpha,
                filters=self._filters,
            )
        )
```

---

### 5.4.3 其他 Retriever

| Retriever | 文件 | 核心逻辑 |
|-----------|------|----------|
| **AutoMergingRetriever** | `retrievers/auto_merging_retriever.py` | 将子节点合并为父节点（基于比例阈值） |
| **QueryFusionRetriever** | `retrievers/fusion_retriever.py` | 多查询 + 多检索器 + RRF 融合 |
| **RecursiveRetriever** | `retrievers/recursive_retriever.py` | 跟随 IndexNode 链接递归检索 |
| **RouterRetriever** | `retrievers/router_retriever.py` | 使用 BaseSelector 选择 N 个检索器之一 |
| **TransformRetriever** | `retrievers/transform_retriever.py` | 检索前应用查询转换 |

---

## 5.5 QueryEngine 层次

### 5.5.1 BaseQueryEngine（`base/base_query_engine.py`）

**文件路径**: `llama-index-core/llama_index/core/base/base_query_engine.py`
**总行数**: 93 行

```python
class BaseQueryEngine(PromptMixin, DispatcherSpanMixin):
    def query(self, str_or_query_bundle) -> RESPONSE_TYPE:
        if isinstance(str_or_query_bundle, str):
            query_bundle = QueryBundle(query_str=str_or_query_bundle)
        with self.callback_manager.event(CBEventType.QUERY, ...) as query_event:
            response = self._query(query_bundle)
            query_event.on_end(payload={EventPayload.RESPONSE: response})
        return response

    async def aquery(self, str_or_query_bundle) -> RESPONSE_TYPE: ...

    @abstractmethod
    def _query(self, query_bundle) -> RESPONSE_TYPE: ...
```

---

### 5.5.2 RetrieverQueryEngine（`query_engine/retriever_query_engine.py`）

**文件路径**: `llama-index-core/llama_index/core/query_engine/retriever_query_engine.py`
**总行数**: ~240 行

#### 类定义

```python
class RetrieverQueryEngine(BaseQueryEngine):
    def __init__(self, retriever, response_synthesizer=None, node_postprocessors=None, callback_manager=None):
        self._retriever = retriever
        self._response_synthesizer = response_synthesizer or get_response_synthesizer(...)
        self._node_postprocessors = node_postprocessors or []
```

#### 核心方法

**`_query(query_bundle)`** (行 202-215):
```python
@dispatcher.span
def _query(self, query_bundle) -> RESPONSE_TYPE:
    with self.callback_manager.event(CBEventType.QUERY, ...) as query_event:
        nodes = self.retrieve(query_bundle)
        response = self._response_synthesizer.synthesize(query=query_bundle, nodes=nodes)
        query_event.on_end(payload={EventPayload.RESPONSE: response})
    return response
```

**`retrieve(query_bundle)`** (行 160-162):
```python
def retrieve(self, query_bundle) -> List[NodeWithScore]:
    nodes = self._retriever.retrieve(query_bundle)
    return self._apply_node_postprocessors(nodes, query_bundle=query_bundle)
```

**`from_args(...)`** (行 62-140):
工厂方法，支持丰富的参数配置：
- `response_mode`: COMPACT / TREE_SUMMARIZE / REFINE / SIMPLE_SUMMARIZE / NO_TEXT / ACCUMULATE / COMPACT_ACCUMULATE
- `text_qa_template` / `refine_template` / `summary_template`: 自定义模板
- `output_cls`: 结构化输出类
- `streaming`: 是否流式
- `multimodal`: 是否多模态

---

### 5.5.3 其他 QueryEngine

| QueryEngine | 核心逻辑 |
|-------------|----------|
| **SubQuestionQueryEngine** | 将复杂问题分解为子问题，分别查询后合并 |
| **FLAREInstructQueryEngine** | 前瞻式检索：生成 → 检查置信度 → 不确定时检索 → 重新生成 |
| **RouterQueryEngine** | 从多个 QueryEngine 中选择一个执行 |
| **RetryQueryEngine** | 失败后重试（基于反馈） |
| **CitationQueryEngine** | 为回答添加引用标注 |
| **MultiStepQueryEngine** | 多步查询（基于计划） |
| **SQLJoinQueryEngine** | SQL + 向量检索联合 |
| **KnowledgeGraphQueryEngine** | 知识图谱查询 |
| **PandasQueryEngine** | Pandas DataFrame 查询 |
| **JSONalyzeQueryEngine** | JSON 数据分析 |

---

## 5.6 设计模式总结

| 模式 | 应用位置 | 效果 |
|------|----------|------|
| **Template Method** | BaseLLM.chat() → _chat() | 子类只需实现核心逻辑 |
| **Strategy** | Retriever / Synthesizer / NodeParser | 运行时替换算法 |
| **Decorator** | StructuredLLM 包装 LLM | 增加功能不改变接口 |
| **Factory** | get_response_synthesizer() / get_program_for_llm() | 根据参数创建实例 |
| **Composite** | RetrieverQueryEngine 组合 Retriever + Synthesizer | 统一接口处理部分-整体 |
| **Observer** | CallbackManager + Dispatcher | 事件通知解耦 |
| **Mixin** | PromptMixin | 复用 Prompt 管理逻辑 |
| **Adapter** | bridge/langchain.py | 适配外部框架类型 |

---

## 5.7 性能瓶颈分析

| 瓶颈位置 | 原因 | 优化建议 |
|----------|------|----------|
| **Embedding 计算** | 大批量文本需逐批计算 | 使用 `get_text_embedding_batch()` + 缓存 |
| **LLM 调用** | 网络延迟 + 速率限制 | 使用 `RateLimiter` + 异步并发 |
| **向量检索** | 大规模向量 ANN 搜索 | 选择高性能 VectorStore（Milvus/Qdrant） |
| **NodeParser** | 大文档切分耗时 | 使用 `num_workers` 并行 |
| **序列化** | `to_dict()` / `from_dict()` 大对象 | 使用增量持久化 |

---

## 5.8 小结

本章深入走读了 LlamaIndex 最核心的 5 个模块：

1. **LLM 抽象层**: 8 个核心方法（4 同步 + 4 异步），`predict()` 自动路由 chat/complete
2. **Embedding 抽象层**: 批量计算 + 缓存 + 速率限制
3. **BaseIndex**: 三种转换方法（as_retriever/as_query_engine/as_chat_engine）
4. **VectorStoreIndex**: 批量嵌入 + 向量存储 + docstore 同步
5. **RetrieverQueryEngine**: 检索 + 后处理 + 合成的经典管线

在下一章中，我们将继续深入 Ingestion Pipeline、Workflow、Agent、Storage、Callback、Prompt、Schema 等模块。

---

# ❤️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../res/promotion.jpg)