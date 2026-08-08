# 第 8 章：API 与接口设计

> 本章详细描述 LlamaIndex 的所有对外及内部 API，包括请求/响应示例、参数说明、认证、限流、版本控制等。

---

## 8.1 API 设计原则

LlamaIndex 的 API 设计遵循以下原则：

1. **分层抽象**: 高级 API（5 行代码）+ 低级 API（完全定制）
2. **异步优先**: 所有 IO 操作提供 sync + async 双版本
3. **类型安全**: 全面使用 Pydantic v2 类型校验
4. **流式支持**: 支持 Token 级流式输出
5. **可组合**: 组件可以自由组合（Index → Retriever → QueryEngine）

---

## 8.2 核心 API 列表

### 8.2.1 Index API

#### VectorStoreIndex

```python
class VectorStoreIndex(BaseIndex[IndexDict]):
    @classmethod
    def from_documents(
        cls,
        documents: Sequence[Document],
        storage_context: Optional[StorageContext] = None,
        show_progress: bool = False,
        callback_manager: Optional[CallbackManager] = None,
        transformations: Optional[List[TransformComponent]] = None,
        **kwargs: Any,
    ) -> VectorStoreIndex:
        """从文档创建索引"""

    def as_retriever(self, **kwargs) -> BaseRetriever:
        """转换为检索器"""

    def as_query_engine(
        self,
        llm: Optional[LLM] = None,
        response_mode: ResponseMode = ResponseMode.COMPACT,
        **kwargs,
    ) -> BaseQueryEngine:
        """转换为查询引擎"""

    def as_chat_engine(
        self,
        chat_mode: ChatMode = ChatMode.BEST,
        **kwargs,
    ) -> BaseChatEngine:
        """转换为对话引擎"""

    def insert(self, document: Document) -> None:
        """插入单个文档"""

    def insert_nodes(self, nodes: Sequence[BaseNode]) -> None:
        """插入节点"""

    def delete_ref_doc(self, ref_doc_id: str, raise_error: bool = True) -> None:
        """删除引用文档"""
```

**请求示例**:
```python
from llama_index.core import VectorStoreIndex, SimpleDirectoryReader

# 从文档创建
documents = SimpleDirectoryReader("data").load_data()
index = VectorStoreIndex.from_documents(documents)

# 转换为查询引擎
query_engine = index.as_query_engine(
    llm=OpenAI(model="gpt-4"),
    response_mode="compact",
    similarity_top_k=5,
)

# 增量插入
index.insert(new_document)
```

### 8.2.2 LLM API

```python
class LLM(BaseLLM):
    # ---- 核心方法 ----
    def chat(self, messages: Sequence[ChatMessage], **kwargs) -> ChatResponse:
        """同步对话"""

    async def achat(self, messages: Sequence[ChatMessage], **kwargs) -> ChatResponse:
        """异步对话"""

    def complete(self, prompt: str, formatted: bool = False, **kwargs) -> CompletionResponse:
        """同步补全"""

    async def acomplete(self, prompt: str, formatted: bool = False, **kwargs) -> CompletionResponse:
        """异步补全"""

    def stream_chat(self, messages, **kwargs) -> ChatResponseGen:
        """同步流式对话"""

    async def astream_chat(self, messages, **kwargs) -> ChatResponseAsyncGen:
        """异步流式对话"""

    # ---- 便捷方法 ----
    def predict(self, prompt: BasePromptTemplate, **kwargs) -> str:
        """自动选择 chat/complete"""

    def stream(self, prompt: BasePromptTemplate, **kwargs) -> TokenGen:
        """流式 predict"""

    # ---- 结构化输出 ----
    def structured_predict(self, output_cls, prompt, **kwargs) -> BaseModel:
        """结构化预测"""

    async def astructured_predict(self, output_cls, prompt, **kwargs) -> BaseModel:
        """异步结构化预测"""

    def stream_structured_predict(self, output_cls, prompt, **kwargs) -> Generator:
        """流式结构化预测"""

    # ---- 工具调用 ----
    def predict_and_call(self, tools, user_msg, **kwargs) -> AgentChatResponse:
        """预测并调用工具（创建 ReActAgent）"""

    # ---- 转换方法 ----
    def as_structured_llm(self, output_cls) -> StructuredLLM:
        """包装为结构化 LLM"""
```

**请求示例**:
```python
from llama_index.llms.openai import OpenAI
from llama_index.core.base.llms.types import ChatMessage, MessageRole

llm = OpenAI(model="gpt-4", temperature=0.7)

# 对话
response = llm.chat([
    ChatMessage(role=MessageRole.SYSTEM, content="You are a helpful assistant."),
    ChatMessage(role=MessageRole.USER, content="What is RAG?")
])
print(response.message.content)

# 流式对话
stream = llm.stream_chat([...])
for chunk in stream:
    print(chunk.delta, end="")

# 结构化输出
from pydantic import BaseModel

class Answer(BaseModel):
    question: str
    answer: str
    confidence: float

output = llm.structured_predict(Answer, PromptTemplate("Answer: {query}"), query="What is RAG?")
print(output.answer, output.confidence)
```

### 8.2.3 QueryEngine API

```python
class BaseQueryEngine(PromptMixin, DispatcherSpanMixin):
    def query(self, str_or_query_bundle) -> RESPONSE_TYPE:
        """同步查询"""

    async def aquery(self, str_or_query_bundle) -> RESPONSE_TYPE:
        """异步查询"""


class RetrieverQueryEngine(BaseQueryEngine):
    @classmethod
    def from_args(
        cls,
        retriever: BaseRetriever,
        llm: Optional[LLM] = None,
        response_mode: ResponseMode = ResponseMode.COMPACT,
        text_qa_template: Optional[BasePromptTemplate] = None,
        refine_template: Optional[BasePromptTemplate] = None,
        node_postprocessors: Optional[List[BaseNodePostprocessor]] = None,
        output_cls: Optional[Type[BaseModel]] = None,
        streaming: bool = False,
        multimodal: bool = False,
        **kwargs,
    ) -> RetrieverQueryEngine:
        """工厂方法创建查询引擎"""

    def retrieve(self, query_bundle) -> List[NodeWithScore]:
        """仅检索（不合成）"""

    def synthesize(self, query_bundle, nodes) -> RESPONSE_TYPE:
        """仅合成（不检索）"""
```

**请求示例**:
```python
# 查询
response = query_engine.query("What is RAG?")
print(response.response)  # 回答文本
print(response.source_nodes)  # 来源节点
print(response.metadata)  # 元数据

# 异步查询
response = await query_engine.aquery("What is RAG?")

# 流式查询
query_engine = index.as_query_engine(streaming=True)
response = query_engine.query("What is RAG?")
for token in response.response_gen:
    print(token, end="")
```

### 8.2.4 Retriever API

```python
class BaseRetriever(PromptMixin, DispatcherSpanMixin):
    def retrieve(self, str_or_query_bundle) -> List[NodeWithScore]:
        """同步检索"""

    async def aretrieve(self, str_or_query_bundle) -> List[NodeWithScore]:
        """异步检索"""


class VectorIndexRetriever(BaseRetriever):
    def __init__(
        self,
        index: VectorStoreIndex,
        similarity_top_k: int = 1,
        node_ids: Optional[List[str]] = None,
        doc_ids: Optional[List[str]] = None,
        filters: Optional[MetadataFilters] = None,
        mode: VectorStoreQueryMode = VectorStoreQueryMode.DEFAULT,
        alpha: Optional[float] = None,
        **kwargs,
    ):
        ...
```

**请求示例**:
```python
# 基础检索
retriever = index.as_retriever(similarity_top_k=5)
nodes = retriever.retrieve("What is RAG?")

for node in nodes:
    print(f"Score: {node.score}, Text: {node.node.text[:100]}")

# 带过滤器的检索
from llama_index.core.vector_stores.types import MetadataFilter, FilterOperator, MetadataFilters

filters = MetadataFilters(
    filters=[
        MetadataFilter(key="author", value="John", operator=FilterOperator.EQ),
        MetadataFilter(key="year", value=2024, operator=FilterOperator.GTE),
    ]
)
retriever = index.as_retriever(filters=filters)
```

### 8.2.5 Agent API

```python
class BaseWorkflowAgent(Workflow, BaseModel, PromptMixin):
    def run(
        self,
        user_msg: Optional[Union[str, ChatMessage]] = None,
        chat_history: Optional[List[ChatMessage]] = None,
        memory: Optional[BaseMemory] = None,
        max_iterations: int = DEFAULT_MAX_ITERATIONS,
        **kwargs,
    ) -> WorkflowHandler:
        """运行 Agent"""

    async def arun(self, user_msg=None, **kwargs) -> WorkflowHandler:
        """异步运行 Agent"""


class AgentWorkflow(Workflow, PromptMixin):
    @classmethod
    def from_tools_or_functions(
        cls,
        tools: List[Union[BaseTool, Callable]],
        llm: LLM,
        ...,
    ) -> AgentWorkflow:
        """从工具创建多 Agent 工作流"""
```

**请求示例**:
```python
from llama_index.core.agent.workflow import ReActAgent, AgentWorkflow
from llama_index.core.tools import FunctionTool

# 定义工具
def search(query: str) -> str:
    """搜索工具"""
    return f"Results for: {query}"

tool = FunctionTool.from_defaults(fn=search)

# 创建 Agent
agent = ReActAgent(
    name="SearchAgent",
    tools=[tool],
    llm=OpenAI(model="gpt-4"),
    system_prompt="You are a search assistant.",
)

# 运行
result = agent.run("Search for LlamaIndex")
print(result)

# 流式运行
handler = agent.run("Search for LlamaIndex")
async for event in handler.stream_events():
    print(event)
result = await handler
```

### 8.2.6 Workflow API

```python
class Workflow(BaseModel):
    def run(self, **kwargs) -> WorkflowHandler:
        """运行工作流"""

    async def arun(self, **kwargs) -> WorkflowHandler:
        """异步运行工作流"""


def step(func) -> Callable:
    """步骤装饰器"""
```

**请求示例**:
```python
from llama_index.core.workflow import Workflow, StartEvent, StopEvent, step, Context

class RAGWorkflow(Workflow):
    @step
    async def retrieve(self, ctx: Context, ev: StartEvent) -> StopEvent:
        query = ev.get("query", "")
        # 检索逻辑
        return StopEvent(result=f"Answer for: {query}")

workflow = RAGWorkflow(timeout=30, verbose=True)
result = workflow.run(query="What is RAG?")
```

### 8.2.7 IngestionPipeline API

```python
class IngestionPipeline(BaseModel):
    def run(
        self,
        documents: Optional[List[Document]] = None,
        nodes: Optional[Sequence[BaseNode]] = None,
        show_progress: bool = False,
        cache_collection: Optional[str] = None,
        num_workers: Optional[int] = None,
        **kwargs,
    ) -> Sequence[BaseNode]:
        """运行摄入管线"""

    async def arun(self, **kwargs) -> Sequence[BaseNode]:
        """异步运行摄入管线"""

    def persist(self, persist_dir: str = "./pipeline_storage") -> None:
        """持久化管线"""

    def load(self, persist_dir: str = "./pipeline_storage") -> None:
        """加载管线"""
```

**请求示例**:
```python
from llama_index.core.ingestion import IngestionPipeline
from llama_index.core.node_parser import SentenceSplitter
from llama_index.embeddings.openai import OpenAIEmbedding

pipeline = IngestionPipeline(
    transformations=[
        SentenceSplitter(chunk_size=512, chunk_overlap=20),
        OpenAIEmbedding(),
    ],
    vector_store=ChromaVectorStore(),
    docstore=SimpleDocumentStore(),
    docstore_strategy=DocstoreStrategy.UPSERTS,
)

nodes = pipeline.run(documents=documents, num_workers=4)
```

### 8.2.8 Settings API

```python
class Settings:
    llm: LLM  # 全局 LLM
    embed_model: BaseEmbedding  # 全局 Embedding
    callback_manager: CallbackManager  # 全局回调管理器
    node_parser: NodeParser  # 全局节点解析器
    transformations: List[TransformComponent]  # 全局转换链
    tokenizer: Callable  # 全局分词器
```

**使用示例**:
```python
from llama_index.core import Settings
from llama_index.llms.openai import OpenAI
from llama_index.embeddings.openai import OpenAIEmbedding

# 全局配置
Settings.llm = OpenAI(model="gpt-4")
Settings.embed_model = OpenAIEmbedding()
Settings.node_parser = SentenceSplitter(chunk_size=512)
```

---

## 8.3 响应类型

### 8.3.1 Response

```python
class Response(BaseModel):
    response: Optional[str]  # 回答文本
    source_nodes: List[NodeWithScore]  # 来源节点
    metadata: Optional[Dict[str, Any]]  # 元数据
```

### 8.3.2 StreamingResponse

```python
class StreamingResponse(BaseModel):
    response_gen: TokenGen  # 流式生成器
    source_nodes: List[NodeWithScore]
    metadata: Optional[Dict[str, Any]]
```

### 8.3.3 ChatResponse

```python
class ChatResponse(BaseModel):
    message: ChatMessage  # 响应消息
    raw: Optional[Any]  # 原始响应
    delta: Optional[str]  # 流式增量
    additional_kwargs: Dict[str, Any]  # 附加参数
```

### 8.3.4 AgentChatResponse

```python
class AgentChatResponse(BaseModel):
    response: str  # 最终回答
    sources: List[ToolOutput]  # 工具输出
    source_nodes: List[NodeWithScore]  # 来源节点
    metadata: Optional[Dict[str, Any]]
```

---

## 8.4 认证与安全

### 8.4.1 LLM 认证

LlamaIndex 不直接管理认证，而是委托给各 LLM SDK：

```python
# OpenAI - 通过环境变量 OPENAI_API_KEY
from llama_index.llms.openai import OpenAI
llm = OpenAI(api_key="sk-...")  # 或从环境变量读取

# Anthropic - 通过环境变量 ANTHROPIC_API_KEY
from llama_index.llms.anthropic import Anthropic
llm = Anthropic(api_key="sk-ant-...")

# 本地模型 - 无需认证
from llama_index.llms.llamafile import Llamafile
llm = Llamafile(base_url="http://localhost:8080")
```

### 8.4.2 VectorStore 认证

```python
# Pinecone
from llama_index.vector_stores.pinecone import PineconeVectorStore
vs = PineconeVectorStore(api_key="...", index_name="my-index")

# Qdrant
from llama_index.vector_stores.qdrant import QdrantVectorStore
vs = QdrantVectorStore(url="http://localhost:6333", api_key="...")
```

---

## 8.5 限流与重试

### 8.5.1 RateLimiter

```python
from llama_index.core.rate_limiter import RateLimiter

# 创建速率限制器（每秒 10 个请求）
rate_limiter = RateLimiter(rate_limit=10, time_period=1.0)

# 应用到 LLM
class RateLimitedLLM(BaseLLM):
    _rate_limiter: RateLimiter
    
    def chat(self, messages, **kwargs):
        self._rate_limiter.acquire(1)  # 获取许可
        return self._llm.chat(messages, **kwargs)
```

### 8.5.2 自动重试

LLM 集成通常使用 `tenacity` 进行自动重试：

```python
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(stop=stop_after_attempt(3), wait=wait_exponential(multiplier=1, min=4, max=10))
def call_llm():
    return client.chat.completions.create(...)
```

---

## 8.6 版本控制

### 8.6.1 语义化版本

LlamaIndex 使用语义化版本 `MAJOR.MINOR.PATCH`：

| 版本 | 说明 |
|------|------|
| **MAJOR** | 不兼容的 API 变更 |
| **MINOR** | 向后兼容的功能新增 |
| **PATCH** | 向后兼容的 Bug 修复 |

### 8.6.2 当前版本

- `llama-index` (元包): v0.14.23
- `llama-index-core`: v0.14.23
- 集成包: 各自独立版本（如 `llama-index-llms-openai` v0.7.x）

### 8.6.3 弃用策略

```python
from deprecated import deprecated

@deprecated(version="0.14.0", reason="Use new_method() instead")
def old_method():
    """此方法已弃用，将在 v0.15.0 中移除"""
    pass
```

---

## 8.7 错误处理

### 8.7.1 异常层次

```
Exception
├── ValueError (参数错误)
├── TypeError (类型错误)
├── WorkflowRuntimeError (工作流运行时错误)
│   ├── WorkflowTimeoutError (超时)
│   └── WorkflowValidationError (验证失败)
└── httpx.HTTPStatusError (HTTP 错误)
```

### 8.7.2 错误处理示例

```python
from llama_index.core.workflow import WorkflowRuntimeError, WorkflowTimeoutError

try:
    result = agent.run("Complex task", max_iterations=50)
except WorkflowTimeoutError:
    print("Agent 执行超时")
except WorkflowRuntimeError as e:
    print(f"Agent 执行错误: {e}")
```

---

## 8.8 内部 API

### 8.8.1 序列化 API

```python
# 序列化
data = component.model_dump()
json_str = component.model_dump_json()

# 反序列化
component = ComponentClass.model_validate(data)
component = ComponentClass.model_validate_json(json_str)

# 从字典加载（通过 class_name）
from llama_index.core.schema import BaseComponent
component = BaseComponent.from_dict(data, class_name=data["class_name"])
```

### 8.8.2 回调 API

```python
from llama_index.core.callbacks import CBEventType, EventPayload

# 注册全局回调
from llama_index.core import set_global_handler
set_global_handler("simple")

# 自定义回调
from llama_index.core.callbacks import BaseCallbackHandler

class MyHandler(BaseCallbackHandler):
    def on_event_start(self, event_type, payload=None, **kwargs):
        print(f"Event started: {event_type}")
    
    def on_event_end(self, event_type, payload=None, **kwargs):
        print(f"Event ended: {event_type}")
```

### 8.8.3 Instrumentation API

```python
import llama_index.core.instrumentation as instrument

# 获取 dispatcher
dispatcher = instrument.get_dispatcher(__name__)

# 手动发出事件
dispatcher.event(MyCustomEvent(data="..."))

# 使用 span 装饰器
@dispatcher.span
async def my_function():
    pass
```

---

## 8.9 小结

本章详细描述了 LlamaIndex 的 API 设计：

1. **7 大核心 API**: Index / LLM / QueryEngine / Retriever / Agent / Workflow / IngestionPipeline
2. **分层设计**: 高级 API + 低级 API
3. **异步优先**: 所有 IO 操作提供 sync + async
4. **类型安全**: Pydantic v2 全面校验
5. **流式支持**: Token 级流式输出
6. **安全**: 认证委托给 SDK，限流通过 RateLimiter

在下一章中，我们将分析部署、运维与基础设施。

---

☕️ 制作不易，请我喝咖啡☕️关注我➕