# 第 6 章：核心代码讲解（下）

> 本章继续深入走读 LlamaIndex 的核心模块：Ingestion Pipeline、Workflow 系统、Agent 系统、Storage 层、Callback & Instrumentation、Prompt 系统、Schema 系统。

---

## 6.1 Ingestion Pipeline（`ingestion/pipeline.py`）

**文件路径**: `llama-index-core/llama_index/core/ingestion/pipeline.py`
**总行数**: ~750 行
**核心类**: `IngestionPipeline`

### 6.1.1 类定义

```python
class IngestionPipeline(BaseModel):
    name: str = Field(default=DEFAULT_PIPELINE_NAME)
    project_name: str = Field(default=DEFAULT_PROJECT_NAME)
    transformations: List[TransformComponent]
    documents: Optional[Sequence[Document]]
    readers: Optional[List[ReaderConfig]]
    vector_store: Optional[BasePydanticVectorStore]
    cache: IngestionCache = Field(default_factory=IngestionCache)
    docstore: Optional[BaseDocumentStore]
    docstore_strategy: DocstoreStrategy = Field(default=DocstoreStrategy.UPSERTS)
    disable_cache: bool = Field(default=False)
```

### 6.1.2 核心方法详解

**`run(show_progress, documents, nodes, cache_collection, in_place, store_doc_text, num_workers, **kwargs)`** (行 538-662):

```python
@dispatcher.span
def run(self, show_progress=False, documents=None, nodes=None,
        cache_collection=None, in_place=True, store_doc_text=True,
        num_workers=None, **kwargs) -> Sequence[BaseNode]:
    # 1. 准备输入
    input_nodes = self._prepare_inputs(documents, nodes)

    # 2. 确定去重策略
    effective_strategy = self.docstore_strategy
    if self.docstore is not None and self.vector_store is None:
        effective_strategy = DocstoreStrategy.DUPLICATES_ONLY

    # 3. 去重
    if self.docstore is not None and self.vector_store is not None:
        if effective_strategy in (UPSERTS, UPSERTS_AND_DELETE):
            nodes_to_run = self._handle_upserts(input_nodes)
        elif effective_strategy == DUPLICATES_ONLY:
            nodes_to_run = self._handle_duplicates(input_nodes)
    # ...

    # 4. 执行转换（并行或串行）
    if num_workers and num_workers > 1:
        # ProcessPoolExecutor 并行
        with multiprocessing.get_context("spawn").Pool(num_workers) as p:
            node_batches = self._node_batcher(num_batches=num_workers, nodes=nodes_to_run)
            worker_results = p.starmap(_run_transformations_worker, ...)
    else:
        nodes = run_transformations(nodes_to_run, self.transformations, ...)

    # 5. 写入向量存储
    if self.vector_store is not None:
        nodes_with_embeddings = [n for n in nodes if n.embedding is not None]
        if nodes_with_embeddings:
            self.vector_store.add(nodes_with_embeddings)

    # 6. 更新文档存储
    if self.docstore is not None:
        self._update_docstore(nodes_to_run, effective_strategy, store_doc_text)

    return nodes
```

### 6.1.3 去重策略详解

**`DocstoreStrategy.UPSERTS`** (`_handle_upserts`, 行 469-507):
- 按 `ref_doc_id` 检查哈希
- 不存在 → 新增
- 哈希变化 → 删除旧版 + 新增
- 相同 → 跳过

**`DocstoreStrategy.DUPLICATES_ONLY`** (`_handle_duplicates`, 行 451-467):
- 仅按哈希去重
- 不处理更新场景

**`DocstoreStrategy.UPSERTS_AND_DELETE`** (在 UPSERTS 基础上):
- 删除本次未出现的旧文档

### 6.1.4 并行执行机制

**`_node_batcher(num_batches, nodes)`** (行 510-516):
```python
@staticmethod
def _node_batcher(num_batches, nodes):
    batch_size = max(1, int(len(nodes) / num_batches))
    for i in range(0, len(nodes), batch_size):
        yield nodes[i : i + batch_size]
```

**`_run_transformations_worker`** (行 186-211):
- 多进程 worker 函数
- 返回 `(nodes, cache_entries)` 元组
- 主进程合并各 worker 的缓存

### 6.1.5 缓存机制

**`get_transformation_hash(nodes, transformation)`** (行 58-69):
```python
def get_transformation_hash(nodes, transformation):
    nodes_str = "".join([str(node.get_content(metadata_mode=MetadataMode.ALL)) for node in nodes])
    transformation_dict = transformation.to_dict()
    transform_string = remove_unstable_values(str(transformation_dict))
    return sha256((nodes_str + transform_string).encode("utf-8")).hexdigest()
```

**设计要点**:
- 使用 SHA256 哈希确保唯一性
- `remove_unstable_values()` 移除不稳定值（如内存地址）
- 缓存命中则跳过转换执行

### 6.1.6 持久化

**`persist(persist_dir, fs)`** (行 374-394):
```python
def persist(self, persist_dir="./pipeline_storage", fs=None, ...):
    self.cache.persist(cache_path, fs=fs)
    if self.docstore is not None:
        self.docstore.persist(docstore_path, fs=fs)
```

**`load(persist_dir, fs)`** (行 396-421):
从磁盘恢复 pipeline 状态（cache + docstore）。

---

## 6.2 Workflow 系统（`workflow/`）

**文件路径**: `llama-index-core/llama_index/core/workflow/`
**说明**: 核心包中的 `workflow/` 是导出层，实际引擎来自外部 `llama-index-workflows` 包。

### 6.2.1 导出内容（`__init__.py`）

```python
from .context import Context
from .context_serializers import JsonPickleSerializer, JsonSerializer
from .decorators import step
from .errors import WorkflowRuntimeError, WorkflowTimeoutError, WorkflowValidationError
from .events import Event, HumanResponseEvent, InputRequiredEvent, StartEvent, StopEvent
from .workflow import Workflow

__all__ = [
    "Context", "Event", "StartEvent", "StopEvent", "Workflow",
    "step", "InputRequiredEvent", "HumanResponseEvent",
    "JsonPickleSerializer", "JsonSerializer",
    "WorkflowRuntimeError", "WorkflowTimeoutError", "WorkflowValidationError",
]
```

### 6.2.2 核心概念

| 概念 | 类 | 说明 |
|------|-----|------|
| **Workflow** | `Workflow` | 工作流引擎，注册步骤，管理事件路由 |
| **Event** | `Event` | 事件基类（Pydantic BaseModel） |
| **StartEvent** | `StartEvent` | 工作流起点事件 |
| **StopEvent** | `StopEvent` | 工作流终点事件 |
| **Context** | `Context` | 工作流上下文（状态共享） |
| **Step** | `@step` | 步骤装饰器 |
| **Handler** | `WorkflowHandler` | 工作流执行句柄 |

### 6.2.3 Step 装饰器（`decorators.py`）

```python
def step(*args, **kwargs) -> Callable:
    kwargs.pop("pass_context", None)  # 移除旧参数
    return upstream_step(*args, **kwargs)
```

LlamaIndex 的 `step` 是对 `llama-index-workflows` 的 `step` 的包装，移除了已弃用的 `pass_context` 参数。

### 6.2.4 Context（`context.py`）

```python
class Context:
    store: BaseKVStore  # 键值存储
    is_running: bool
    
    async def collect_events(self, ev, expected):
        """等待多个事件"""
    
    def send_event(self, event):
        """发送事件到指定步骤"""
    
    def write_event_to_stream(self, event):
        """向外部流式输出事件"""
```

### 6.2.5 使用示例

```python
from llama_index.core.workflow import Workflow, StartEvent, StopEvent, step

class MyWorkflow(Workflow):
    @step
    async def start(self, ctx: Context, ev: StartEvent) -> StopEvent:
        name = ev.get("name", "World")
        return StopEvent(result=f"Hello, {name}!")

workflow = MyWorkflow(timeout=10, verbose=True)
result = workflow.run(name="LlamaIndex")
print(result)  # "Hello, LlamaIndex!"
```

---

## 6.3 Agent 系统（`agent/workflow/`）

**文件路径**: `llama-index-core/llama_index/core/agent/workflow/`
**核心类**: `BaseWorkflowAgent`, `ReActAgent`, `FunctionAgent`, `CodeActAgent`, `AgentWorkflow`

### 6.3.1 BaseWorkflowAgent（`base_agent.py`）

**继承链**: `Workflow + BaseModel + PromptMixin → BaseWorkflowAgent`
**元类**: `BaseWorkflowAgentMeta(WorkflowMeta, ModelMetaclass)`

#### 关键字段

```python
class BaseWorkflowAgent(Workflow, BaseModel, PromptMixin):
    name: str = "Agent"
    description: str = "An agent that can perform a task"
    system_prompt: Optional[str] = None
    tools: Optional[List[Union[BaseTool, Callable]]] = None
    tool_retriever: Optional[ObjectRetriever] = None
    can_handoff_to: Optional[List[str]] = None
    llm: LLM = Field(default_factory=get_default_llm)
    initial_state: Dict[str, Any] = Field(default_factory=dict)
    state_prompt: Union[str, BasePromptTemplate] = DEFAULT_STATE_PROMPT
    output_cls: Optional[Type[BaseModel]] = None
    structured_output_fn: Optional[Callable] = None
    streaming: bool = True
    early_stopping_method: Literal["force", "generate"] = "force"
```

#### 抽象方法

| 方法 | 说明 |
|------|------|
| `take_step(ctx, llm_input, tools, memory) → AgentOutput` | 执行单步推理 |
| `handle_tool_call_results(ctx, results, memory)` | 处理工具调用结果 |
| `finalize(ctx, output, memory) → AgentOutput` | 生成最终输出 |

#### Workflow 步骤（@step 装饰的方法）

1. **`init_run(ctx, ev: AgentWorkflowStartEvent) → AgentInput`** (行 383-435):
   - 初始化 memory、state、max_iterations
   - 处理 user_msg / chat_history
   - 将消息添加到 memory

2. **`setup_agent(ctx, ev: AgentInput) → AgentSetup`** (行 437-463):
   - 拼接 system_prompt
   - 格式化 state 到消息中

3. **`run_agent_step(ctx, ev: AgentSetup) → AgentOutput`** (行 465-480):
   - 调用抽象方法 `take_step()`

4. **`parse_agent_output(ctx, ev: AgentOutput) → StopEvent | AgentInput | ToolCall | None`** (行 520-622):
   - 检查 max_iterations
   - 无 tool_calls → finalize → StopEvent
   - 有 tool_calls → 发送 ToolCall 事件

5. **`call_tool(ctx, ev: ToolCall) → ToolCallResult`** (行 624-659):
   - 查找并执行工具
   - 捕获异常并包装为错误输出

6. **`aggregate_tool_results(ctx, ev: ToolCallResult) → AgentInput | StopEvent | None`** (行 661-720+):
   - 收集所有工具结果
   - 处理 return_direct / handoff
   - 回到 setup_agent 继续循环

#### 关键设计点

**`_get_llm_response(ctx, llm_input, llm)`** (行 314-345):
- 如果 `streaming=True`，使用 `astream_chat()` 并实时发出 `AgentStream` 事件
- 支持 `thinking_delta`（思考过程流式输出）

**`_call_tool(ctx, tool, tool_input)`** (行 347-381):
- 处理 `requires_context` 的工具（注入 ctx）
- 异常捕获并包装为 `ToolOutput(is_error=True)`

**Early Stopping** (行 482-518):
- `early_stopping_method="force"`: 达到 max_iterations 时抛出 `WorkflowRuntimeError`
- `early_stopping_method="generate"`: 生成最终响应而非报错

---

### 6.3.2 ReActAgent（`react_agent.py`）

**继承**: `BaseWorkflowAgent → ReActAgent`

```python
class ReActAgent(BaseWorkflowAgent):
    async def take_step(self, ctx, llm_input, tools, memory) -> AgentOutput:
        # 1. 格式化聊天历史 + 工具
        chat_formatter = ReActChatFormatter(...)
        formatted_messages = chat_formatter.format(tools, llm_input, current_reasoning)
        
        # 2. 调用 LLM
        chat_response = await self._get_llm_response(ctx, formatted_messages)
        
        # 3. 解析输出
        reasoning_steps, is_done = self._output_parser.parse(chat_response.message.content)
        
        # 4. 更新推理历史
        current_reasoning.extend(reasoning_steps)
        
        # 5. 构建 AgentOutput
        if is_done:
            return AgentOutput(response=reasoning_steps[-1], tool_calls=[], ...)
        else:
            tool_calls = [ToolSelection(tool_name=step.tool, tool_kwargs=step.input, ...) 
                         for step in reasoning_steps if step.is_tool_call]
            return AgentOutput(response=..., tool_calls=tool_calls, ...)
```

**ReAct 循环**:
1. Thought: LLM 生成思考
2. Action: LLM 选择工具 + 输入
3. Observation: 执行工具获取结果
4. 重复 1-3 直到 Finish

---

### 6.3.3 FunctionAgent（`function_agent.py`）

**继承**: `BaseWorkflowAgent → FunctionAgent`

```python
class FunctionAgent(BaseWorkflowAgent):
    initial_tool_choice: Optional[str] = None
    allow_parallel_tool_calls: bool = False

    async def take_step(self, ctx, llm_input, tools, memory) -> AgentOutput:
        # 使用 LLM 的函数调用能力
        chat_response = await self._get_llm_response(ctx, llm_input)
        
        # 解析 tool_calls
        tool_calls = chat_response.message.additional_kwargs.get("tool_calls", [])
        
        return AgentOutput(
            response=chat_response.message,
            tool_calls=[ToolSelection(...) for tool_call in tool_calls],
            ...
        )
```

**与 ReActAgent 的区别**:
- ReActAgent: 通过文本解析工具调用
- FunctionAgent: 通过 LLM 原生 Function Calling API

---

### 6.3.4 AgentWorkflow（`multi_agent_workflow.py`）

**继承**: `Workflow + PromptMixin → AgentWorkflow`

```python
class AgentWorkflow(Workflow, PromptMixin):
    agents: Dict[str, BaseWorkflowAgent]
    root_agent: str
    initial_state: Dict[str, Any]
    
    @classmethod
    def from_tools_or_functions(cls, tools, llm, ...):
        """根据工具类型自动选择 FunctionAgent 或 ReActAgent"""
```

**多 Agent 协作**:
- 支持 Agent 间 Handoff（交接）
- 自动创建 `handoff` 工具
- 管理多个 Agent 的状态和上下文

---

## 6.4 Storage 层（`storage/`）

**文件路径**: `llama-index-core/llama_index/core/storage/`
**核心类**: `StorageContext`, `BaseDocumentStore`, `BaseIndexStore`, `BaseKVStore`, `BaseChatStore`

### 6.4.1 StorageContext（`storage_context.py`）

```python
@dataclass
class StorageContext:
    docstore: BaseDocumentStore
    index_store: BaseIndexStore
    vector_stores: Dict[str, SerializeAsAny[BasePydanticVectorStore]]
    graph_store: GraphStore
    property_graph_store: Optional[PropertyGraphStore] = None

    @classmethod
    def from_defaults(cls, docstore=None, index_store=None, vector_store=None,
                      graph_store=None, property_graph_store=None,
                      persist_dir=None, fs=None) -> StorageContext:
        if persist_dir is None:
            docstore = docstore or SimpleDocumentStore()
            index_store = index_store or SimpleIndexStore()
            graph_store = graph_store or SimpleGraphStore()
            vector_stores = {VECTOR_STORE_KEY: vector_store or SimpleVectorStore()}
        else:
            # 从磁盘加载
            docstore = SimpleDocumentStore.from_persist_path(...)
            index_store = SimpleIndexStore.from_persist_path(...)
            # ...
        return cls(docstore=docstore, index_store=index_store, ...)

    def persist(self, persist_dir, fs=None):
        """持久化所有存储到磁盘"""
        self.docstore.persist(...)
        self.index_store.persist(...)
        for vector_store in self.vector_stores.values():
            vector_store.persist(...)
```

**设计要点**:
- `from_defaults()` 工厂方法支持内存模式（默认）和持久化模式
- `vector_stores` 是字典，支持多向量存储（如文本 + 图像）
- `property_graph_store` 延迟初始化

### 6.4.2 BaseDocumentStore（`docstore/`）

```python
class BaseDocumentStore(BaseModel):
    docs: Dict[str, BaseNode] = Field(default_factory=dict)
    
    def add_documents(self, docs, store_text=True, allow_update=False): ...
    def get_document(self, doc_id, raise_error=True) -> Optional[BaseNode]: ...
    def delete_document(self, doc_id, raise_error=True): ...
    def get_all_document_hashes(self) -> Dict[str, str]: ...
    def set_document_hashes(self, doc_hashes): ...
```

**SimpleDocumentStore**: 基于内存字典的实现，支持持久化到 JSON 文件。

### 6.4.3 BaseIndexStore（`index_store/`）

```python
class BaseIndexStore(BaseModel):
    def add_index_struct(self, index_struct): ...
    def get_index_struct(self, struct_id=None) -> Optional[IndexStruct]: ...
    def delete_index_struct(self, struct_id): ...
```

### 6.4.4 BaseKVStore（`kvstore/`）

```python
class BaseKVStore(BaseModel, ABC):
    @abstractmethod
    def put(self, key, val, collection=None): ...
    @abstractmethod
    async def aput(self, key, val, collection=None): ...
    @abstractmethod
    def get(self, key, collection=None): ...
    @abstractmethod
    def get_all(self, collection=None): ...
```

---

## 6.5 Callback & Instrumentation

### 6.5.1 CallbackManager（`callbacks/base.py`）

```python
class CallbackManager(BaseCallbackHandler, ABC):
    def __init__(self, handlers: List[BaseCallbackHandler]):
        self._handlers = handlers
        self._trace_stack = ContextVar("global_stack_trace")
        self._trace_id_stack = ContextVar("global_stack_trace_ids")

    def on_event_start(self, event_type, payload=None, event_id=None, **kwargs):
        """事件开始"""
        for handler in self._handlers:
            handler.on_event_start(event_type, payload, event_id, **kwargs)

    def on_event_end(self, event_type, payload=None, event_id=None, **kwargs):
        """事件结束"""
        for handler in self._handlers:
            handler.on_event_end(event_type, payload, event_id, **kwargs)

    @contextmanager
    def event(self, event_type, payload=None, **kwargs):
        """事件上下文管理器"""
        self.on_event_start(event_type, payload, **kwargs)
        yield EventContext(self, event_type, ...)
        self.on_event_end(event_type, payload, **kwargs)

    @contextmanager
    def as_trace(self, trace_id):
        """追踪上下文"""
        self.start_trace(trace_id)
        yield
        self.end_trace()
```

**事件类型** (`CBEventType`):
- `LLM`: LLM 调用
- `EMBEDDING`: Embedding 计算
- `RETRIEVE`: 检索
- `QUERY`: 查询
- `SYNTHESIZE`: 合成
- `TREE`: 树形摘要
- `SUB_QUESTION`: 子问题
- `AGENT_STEP`: Agent 步骤
- `FUNC_TOOL`: 函数工具调用

### 6.5.2 Instrumentation（`instrumentation/`）

**Dispatcher** (`dispatcher.py`):
```python
class Dispatcher(BaseModel):
    event_handlers: List[BaseEventHandler]
    span_handlers: List[BaseSpanHandler]
    parent_name: Optional[str] = None
    manager: Optional[Manager] = None
    propagate: bool = True

    def event(self, event: BaseEvent):
        """派发事件到所有 handler"""
        for handler in self.event_handlers:
            handler.handle(event)
        if self.propagate and self.parent_name:
            self.manager.dispatch(event)

    def span(self, func):
        """装饰器：包装函数为 Span"""
        @functools.wraps(func)
        async def async_wrapper(*args, **kwargs):
            self.span_enter(...)
            try:
                result = await func(*args, **kwargs)
                self.span_exit(...)
                return result
            except Exception as e:
                self.span_drop(...)
                raise
        return async_wrapper
```

**事件类型** (`events/`):
- `LLMPredictStartEvent` / `LLMPredictEndEvent`
- `EmbeddingStartEvent` / `EmbeddingEndEvent`
- `RetrievalStartEvent` / `RetrievalEndEvent`
- `QueryStartEvent` / `QueryEndEvent`
- `SynthesisStartEvent` / `SynthesisEndEvent`
- `AgentStartEvent` / `AgentEndEvent`
- `RerankStartEvent` / `RerankEndEvent`
- `ExceptionEvent`
- `SpanDropEvent`

---

## 6.6 Prompt 系统（`prompts/`）

**文件路径**: `llama-index-core/llama_index/core/prompts/`
**核心类**: `BasePromptTemplate`, `PromptTemplate`, `ChatPromptTemplate`, `PromptMixin`

### 6.6.1 BasePromptTemplate（`base.py`）

```python
class BasePromptTemplate(BaseModel, ABC):
    metadata: Dict[str, Any] = {}
    template_vars: Optional[List[str]] = None
    kwargs: Dict[str, Any] = {}
    output_parser: Optional[BaseOutputParser] = None
    template_var_mappings: Optional[Dict[str, Any]] = None
    function_mappings: Optional[Dict[str, Callable]] = None

    @abstractmethod
    def format(self, llm=None, messages_to_prompt=None, **kwargs) -> str: ...

    @abstractmethod
    def format_messages(self, llm=None, **kwargs) -> List[ChatMessage]: ...

    def partial_format(self, **kwargs) -> "BasePromptTemplate":
        """部分格式化（返回新模板）"""
```

### 6.6.2 PromptTemplate（`prompts.py`）

```python
class PromptTemplate(BasePromptTemplate):
    template: str  # "请回答关于 {topic} 的问题：{query}"

    def format(self, llm=None, messages_to_prompt=None, **kwargs) -> str:
        template_vars = {**self.kwargs, **kwargs}
        if self.template_var_mappings:
            template_vars = {k: self.template_var_mappings.get(v, v) for k, v in template_vars.items()}
        return self.template.format(**template_vars)
```

### 6.6.3 PromptMixin（`mixin.py`）

```python
class PromptMixin(BaseModel):
    def _get_prompts(self) -> PromptDictType:
        """获取所有 prompt"""
        return {}

    def _update_prompts(self, prompts: PromptDictType):
        """更新 prompt"""
        pass

    def _get_prompt_modules(self) -> PromptMixinType:
        """获取子模块（递归获取 prompt）"""
        return {}

    def get_prompts(self) -> PromptDictType:
        """获取所有 prompt（包括子模块）"""
        prompts = self._get_prompts()
        for module_name, module in self._get_prompt_modules().items():
            if module is not None:
                module_prompts = module.get_prompts()
                prompts.update({f"{module_name}.{k}": v for k, v in module_prompts.items()})
        return prompts
```

**设计要点**:
- `PromptMixin` 让任何组件都可以管理自己的 Prompt
- 支持递归获取子模块的 Prompt
- 支持批量更新 Prompt

---

## 6.7 Schema 系统（`schema.py`）

**文件路径**: `llama-index-core/llama_index/core/schema.py`
**总行数**: 1,492 行（核心包最大文件）

### 6.7.1 BaseComponent

```python
class BaseComponent(BaseModel):
    @classmethod
    def class_name(cls) -> str:
        return "base_component"

    @model_serializer(mode="wrap")
    def custom_model_dump(self, handler, info):
        data = handler(self)
        data["class_name"] = self.class_name()
        return data
```

### 6.7.2 Document / TextNode

```python
class TextNode(BaseNode):
    text: str = ""
    metadata: Dict[str, Any] = {}
    relationships: Dict[NodeRelationship, RelatedNodeInfo] = {}
    hash: Optional[str] = None
    embedding: Optional[List[float]] = None
    excluded_embed_metadata_keys: List[str] = []
    excluded_llm_metadata_keys: List[str] = []
    metadata_separator: str = "\n"
    text_template: str = DEFAULT_TEXT_NODE_TMPL
    metadata_template: str = DEFAULT_METADATA_TMPL

    def get_content(self, metadata_mode=MetadataMode.NONE) -> str:
        """获取内容（可选包含元数据）"""
        if metadata_mode == MetadataMode.NONE:
            return self.text
        metadata_str = self.get_metadata_str(mode=metadata_mode)
        return self.text_template.format(metadata_str=metadata_str, content=self.text)

    def get_metadata_str(self, mode=MetadataMode.ALL) -> str:
        """获取元数据字符串"""
        # 过滤 excluded 的键
        metadata_dict = {k: v for k, v in self.metadata.items()
                        if k not in self.excluded_embed_metadata_keys}
        return self.metadata_separator.join(
            self.metadata_template.format(key=k, value=v) for k, v in metadata_dict.items()
        )
```

### 6.7.3 NodeWithScore

```python
class NodeWithScore(BaseNode):
    node: SerializeAsAny[BaseNode]
    score: Optional[float] = None

    def get_score(self, raise_error=False) -> float:
        if self.score is None:
            if raise_error:
                raise ValueError("Score not set!")
            return 0.0
        return self.score
```

### 6.7.4 QueryBundle

```python
class QueryBundle(BaseModel):
    query_str: str
    custom_embedding_strs: Optional[List[str]] = None
    embedding: Optional[List[float]] = None
    embedding_image_assets: Optional[List[ImageNode]] = None
```

### 6.7.5 ContentBlock 体系

```python
class BaseContentBlock(ABC):
    """内容块基类"""

class TextBlock(BaseContentBlock):
    text: str

class ImageBlock(BaseContentBlock):
    url: Optional[AnyUrl] = None
    image: Optional[bytes] = None  # base64
    image_mimetype: Optional[str] = None
    detail: Optional[str] = None

class AudioBlock(BaseContentBlock):
    url: Optional[AnyUrl] = None
    audio: Optional[bytes] = None
    audio_mimetype: Optional[str] = None

class VideoBlock(BaseContentBlock):
    url: Optional[AnyUrl] = None
    video: Optional[bytes] = None
    video_mimetype: Optional[str] = None

class DocumentBlock(BaseContentBlock):
    data: Optional[bytes] = None
    url: Optional[AnyUrl] = None
    document_mimetype: Optional[str] = None

class ThinkingBlock(BaseContentBlock):
    content: str
    thinking: str

class CachePoint(BaseContentBlock):
    """缓存断点，用于 Prompt Caching"""
    config: Optional[CachePointConfig] = None
```

### 6.7.6 ChatMessage

```python
class ChatMessage(BaseModel):
    role: MessageRole
    blocks: List[BaseContentBlock] = Field(default_factory=list)
    additional_kwargs: Dict[str, Any] = {}
    metadata: Optional[Dict[str, Any]] = None

    @property
    def content(self) -> Optional[str]:
        """便捷属性：获取第一个 TextBlock 的内容"""
        for block in self.blocks:
            if isinstance(block, TextBlock):
                return block.text
        return None
```

---

## 6.8 设计模式总结

| 模式 | 应用位置 | 效果 |
|------|----------|------|
| **Template Method** | BaseWorkflowAgent.take_step() | 子类实现具体推理逻辑 |
| **Strategy** | DocstoreStrategy 枚举 | 运行时选择去重策略 |
| **Observer** | CallbackManager + Dispatcher | 事件通知解耦 |
| **Mixin** | PromptMixin | 复用 Prompt 管理 |
| **Factory** | StorageContext.from_defaults() | 创建存储上下文 |
| **Decorator** | @step, @dispatcher.span | 增强函数功能 |
| **Composite** | StorageContext 组合多个 Store | 统一存储接口 |
| **State Machine** | Agent Workflow 步骤转换 | 管理 Agent 状态 |

---

## 6.9 潜在问题与改进建议

| 模块 | 问题 | 改进建议 |
|------|------|----------|
| **IngestionPipeline** | `num_workers` 使用 spawn 模式，启动开销大 | 考虑使用 fork 模式或线程池 |
| **BaseWorkflowAgent** | `take_step` 是抽象方法，子类必须实现全部 | 提供默认实现（如直接 LLM 调用） |
| **StorageContext** | `vector_stores` 字典的键管理不集中 | 使用枚举定义标准键名 |
| **CallbackManager** | 事件类型是字符串，无编译时检查 | 使用枚举替代字符串 |
| **PromptMixin** | `get_prompts()` 递归可能栈溢出 | 添加深度限制 |
| **schema.py** | 单文件 1492 行，过大 | 拆分为多个子模块 |

---

## 6.10 小结

本章深入走读了 LlamaIndex 的 7 个核心模块：

1. **Ingestion Pipeline**: 转换链 + 去重 + 并行 + 缓存
2. **Workflow**: 事件驱动的工作流引擎
3. **Agent**: 基于 Workflow 的多步推理系统
4. **Storage**: DocStore + IndexStore + KVStore + ChatStore
5. **Callback/Instrumentation**: 双层事件系统
6. **Prompt**: 模板 + Mixin 管理
7. **Schema**: Document/TextNode/NodeWithScore/ContentBlock 体系

在下一章中，我们将分析数据模型与数据库设计。

---

# ❤️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../res/promotion.jpg)