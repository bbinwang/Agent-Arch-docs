# 7. API 与接口设计 (API & Interface Design)

> 本章详细描述 Advanced RAG Techniques 项目的所有对外及内部 API、函数接口、LLM 接口、数据流接口。预计字数：~6000 字。

---

## 7.1 API 设计概述

本项目是**离线运行的代码仓库**，不提供 RESTful API 服务。所有 API 表现为：

1. **Python 函数接口**：`helper_functions.py` 中的共享函数
2. **类接口**：各技术脚本中的 RAG 类
3. **CLI 接口**：通过 `argparse` 暴露的命令行接口
4. **LLM API 调用**：对 OpenAI/Anthropic 等外部服务的调用

---

## 7.2 核心函数 API

### 7.2.1 encode_pdf()

```python
def encode_pdf(
    path: str,                    # PDF 文件路径
    chunk_size: int = 1000,       # 文本块大小（字符数）
    chunk_overlap: int = 200      # 块重叠大小（字符数）
) -> FAISS:                        # 返回 FAISS 向量存储
```

**调用示例**：
```python
vectorstore = encode_pdf(
    path="data/Understanding_Climate_Change.pdf",
    chunk_size=1000,
    chunk_overlap=200
)
```

**异常**：
- `FileNotFoundError`：文件不存在
- `RuntimeError`：编码过程出错

### 7.2.2 retrieve_context_per_question()

```python
def retrieve_context_per_question(
    question: str,                           # 用户查询
    chunks_query_retriever: BaseRetriever     # LangChain 检索器
) -> List[str]:                               # 返回上下文文本列表
```

**调用示例**：
```python
context = retrieve_context_per_question(
    "What is climate change?",
    retriever
)
```

### 7.2.3 answer_question_from_context()

```python
def answer_question_from_context(
    question: str,                    # 用户问题
    context: List[str],               # 检索上下文
    question_answer_from_context_chain: Chain  # 问答链
) -> Dict[str, Any]:                  # 返回 {"answer", "context", "question"}
```

### 7.2.4 bm25_retrieval()

```python
def bm25_retrieval(
    bm25: BM25Okapi,          # BM25 索引
    cleaned_texts: List[str], # 文本列表
    query: str,               # 查询
    k: int = 5                # 返回数量
) -> List[str]:               # Top-k 文本
```

### 7.2.5 retry_with_exponential_backoff()

```python
async def retry_with_exponential_backoff(
    coroutine: Coroutine,     # 异步协程
    max_retries: int = 5      # 最大重试次数
) -> Any:                     # 协程返回值
```

**重试策略**：
- 仅捕获 `RateLimitError`
- 退避公式：`wait = 2^attempt + random(0, 1)`
- 最大重试 5 次

---

## 7.3 类接口 API

### 7.3.1 SimpleRAG 类

```python
class SimpleRAG:
    def __init__(
        self,
        path: str,                # PDF 路径
        chunk_size: int = 1000,   # 块大小
        chunk_overlap: int = 200,  # 块重叠
        n_retrieved: int = 2      # 检索数量
    ):
        self.vector_store = encode_pdf(path, chunk_size, chunk_overlap)
        self.chunks_query_retriever = self.vector_store.as_retriever(
            search_kwargs={"k": n_retrieved}
        )
    
    def run(self, query: str) -> None:
        """执行检索并打印结果"""
        context = retrieve_context_per_question(query, self.chunks_query_retriever)
        show_context(context)
```

### 7.3.2 AdaptiveRAG 类

```python
class AdaptiveRAG:
    def __init__(self, texts: List[str]):
        self.classifier = QueryClassifier()
        self.strategies = {
            "Factual": FactualRetrievalStrategy(texts),
            "Analytical": AnalyticalRetrievalStrategy(texts),
            "Opinion": OpinionRetrievalStrategy(texts),
            "Contextual": ContextualRetrievalStrategy(texts)
        }
    
    def answer(self, query: str) -> str:
        """分类 → 选择策略 → 检索 → 生成答案"""
        category = self.classifier.classify(query)
        strategy = self.strategies[category]
        docs = strategy.retrieve(query)
        return self.llm_chain.invoke({
            "context": "\n".join([d.page_content for d in docs]),
            "question": query
        }).content
```

### 7.3.3 SelfRAG 类

```python
class SelfRAG:
    def __init__(self, path: str, top_k: int = 3):
        self.vectorstore = encode_pdf(path)
        self.top_k = top_k
        # 5 个 LLM Chain
        self.retrieval_chain = ...
        self.relevance_chain = ...
        self.generation_chain = ...
        self.support_chain = ...
        self.utility_chain = ...
    
    def run(self, query: str) -> str:
        """
        6 步流水线：
        1. 检索决策 → 2. 检索 → 3. 相关性评估 → 
        4. 生成 → 5. 支持度评估 → 6. 效用评估
        """
```

### 7.3.4 CRAG 类

```python
class CRAG:
    def __init__(
        self,
        path: str,
        model: str = "gpt-4o-mini",
        max_tokens: int = 1000,
        temperature: float = 0,
        lower_threshold: float = 0.3,
        upper_threshold: float = 0.7
    ):
        self.vectorstore = encode_pdf(path)
        self.llm = ChatOpenAI(model=model, ...)
        self.search = DuckDuckGoSearchResults()
    
    def evaluate_documents(self, query: str, documents: List[str]) -> List[float]:
        """评估文档相关性"""
    
    def knowledge_refinement(self, document: str) -> List[str]:
        """知识精炼"""
    
    def rewrite_query(self, query: str) -> str:
        """查询改写"""
```

### 7.3.5 HyDERetriever 类

```python
class HyDERetriever:
    def __init__(self, files_path: str, chunk_size: int = 500, chunk_overlap: int = 100):
        self.llm = ChatOpenAI(temperature=0, model_name="gpt-4o-mini")
        self.embeddings = OpenAIEmbeddings()
        self.vectorstore = encode_pdf(files_path, chunk_size, chunk_overlap)
    
    def generate_hypothetical_document(self, query: str) -> str:
        """生成假设计答文档"""
    
    def retrieve(self, query: str, k: int = 3) -> Tuple[List[Document], str]:
        """使用假设计答检索"""
```

### 7.3.6 FusionRetrievalRAG 类

```python
class FusionRetrievalRAG:
    def __init__(self, path: str, chunk_size: int = 1000, chunk_overlap: int = 200):
        self.vectorstore, self.cleaned_texts = encode_pdf_and_get_split_documents(...)
        self.bm25 = create_bm25_index(self.cleaned_texts)
    
    def run(self, query: str, k: int = 5, alpha: float = 0.5) -> None:
        """执行混合检索"""
        top_docs = fusion_retrieval(self.vectorstore, self.bm25, query, k, alpha)
```

---

## 7.4 CLI 接口

### 7.4.1 统一 CLI 模式

所有脚本使用 `argparse` 提供统一 CLI 接口：

```bash
python simple_rag.py --path data/doc.pdf --query "question" --chunk_size 1000
```

### 7.4.2 各脚本 CLI 参数

| 脚本 | 特有参数 | 默认值 |
|------|----------|--------|
| `simple_rag.py` | `--evaluate` | False |
| `HyDE.py` | `--chunk_size` | 500 |
| `fusion_retrieval.py` | `--alpha`, `--k` | 0.5, 5 |
| `reranking.py` | `--retriever_type` | "reranker" |
| `crag.py` | `--lower_threshold`, `--upper_threshold` | 0.3, 0.7 |
| `semantic_chunking.py` | `--breakpoint_threshold_type` | "percentile" |

### 7.4.3 CLI 调用示例

```bash
# Simple RAG
python all_rag_techniques_runnable_scripts/simple_rag.py \
    --path data/Understanding_Climate_Change.pdf \
    --query "What is climate change?" \
    --chunk_size 1000 \
    --n_retrieved 3

# HyDE
python all_rag_techniques_runnable_scripts/HyDe_Hypothetical_Document_Embedding.py \
    --path data/Understanding_Climate_Change.pdf \
    --query "What causes climate change?"

# Fusion Retrieval
python all_rag_techniques_runnable_scripts/fusion_retrieval.py \
    --path data/Understanding_Climate_Change.pdf \
    --query "Impacts of climate change" \
    --alpha 0.6 \
    --k 5

# Reranking
python all_rag_techniques_runnable_scripts/reranking.py \
    --path data/Understanding_Climate_Change.pdf \
    --query "Climate change effects" \
    --retriever_type cross_encoder
```

---

## 7.5 LLM API 接口

### 7.5.1 OpenAI API 调用

```python
from langchain_openai import ChatOpenAI, OpenAIEmbeddings

# 聊天模型
llm = ChatOpenAI(
    temperature=0,              # 确定性输出
    model_name="gpt-4o",       # 模型选择
    max_tokens=4000             # 最大输出 token
)

# 嵌入模型
embeddings = OpenAIEmbeddings()  # 默认 text-embedding-3
```

### 7.5.2 结构化输出 API

```python
from pydantic import BaseModel, Field

class Response(BaseModel):
    answer: str = Field(description="The answer")

# LangChain 结构化输出
chain = prompt | llm.with_structured_output(Response)
result = chain.invoke({"query": "..."})
# result.answer 直接可用
```

### 7.5.3 异步 API

```python
import asyncio

async def async_invoke(chain, input_data):
    return await chain.ainvoke(input_data)

# 并发调用
results = await asyncio.gather(
    async_invoke(chain1, input1),
    async_invoke(chain2, input2),
    async_invoke(chain3, input3)
)
```

### 7.5.4 重试 API

```python
from openai import RateLimitError

async def call_with_retry(coroutine, max_retries=5):
    for attempt in range(max_retries):
        try:
            return await coroutine
        except RateLimitError:
            wait = (2 ** attempt) + random.uniform(0, 1)
            await asyncio.sleep(wait)
    raise Exception("Max retries reached")
```

---

## 7.6 评估 API

### 7.6.1 DeepEval API

```python
from deepeval import evaluate
from deepeval.metrics import GEval, FaithfulnessMetric, ContextualRelevancyMetric
from deepeval.test_case import LLMTestCase

# 创建测试用例
test_case = LLMTestCase(
    input="What is climate change?",
    expected_output="Climate change refers to...",
    actual_output="Climate change is...",
    retrieval_context=["context1", "context2"]
)

# 评估
metrics = [
    GEval(name="Correctness", model="gpt-4-turbo", ...),
    FaithfulnessMetric(threshold=0.7, model="gpt-4-turbo"),
    ContextualRelevancyMetric(threshold=1, model="gpt-4-turbo")
]

results = evaluate([test_case], metrics)
```

### 7.6.2 自定义评估 API

```python
def evaluate_rag(retriever, num_questions: int = 5) -> Dict[str, Any]:
    """
    评估 RAG 系统性能
    
    Args:
        retriever: 检索器
        num_questions: 测试问题数量
    
    Returns:
        {
            "questions": [...],
            "results": [...],
            "average_scores": {...}
        }
    """
```

---

## 7.7 数据流接口

### 7.7.1 文档加载接口

```python
from langchain_community.document_loaders import PyPDFLoader

loader = PyPDFLoader("path/to/doc.pdf")
documents = loader.load()  # List[Document]
```

### 7.7.2 文本分割接口

```python
from langchain_text_splitters import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,
    chunk_overlap=200,
    length_function=len
)
chunks = splitter.split_documents(documents)
```

### 7.7.3 向量存储接口

```python
from langchain_community.vectorstores import FAISS

vectorstore = FAISS.from_documents(documents, embeddings)
docs = vectorstore.similarity_search(query, k=5)
docs_with_scores = vectorstore.similarity_search_with_score(query, k=5)
```

### 7.7.4 检索器接口

```python
from langchain_core.retrievers import BaseRetriever

retriever = vectorstore.as_retriever(search_kwargs={"k": 5})
docs = retriever.get_relevant_documents(query)
```

---

## 7.8 接口版本控制

### 7.8.1 当前状态

- **无显式版本控制**：接口通过函数签名隐式版本化
- **向后兼容**：新增参数使用默认值

### 7.8.2 建议版本策略

```python
# 使用 DeprecationWarning
import warnings

def encode_pdf(path, chunk_size=1000, chunk_overlap=200, 
               use_cache=False):  # 新增参数
    if not use_cache:
        warnings.warn(
            "Cache will be enabled by default in v2.0",
            DeprecationWarning
        )
    ...
```

---

## 7.9 认证与安全

### 7.9.1 API Key 管理

```python
# .env 文件
OPENAI_API_KEY=sk-...

# 加载
from dotenv import load_dotenv
load_dotenv()
os.environ["OPENAI_API_KEY"] = os.getenv('OPENAI_API_KEY')
```

### 7.9.2 安全建议

| 方面 | 当前状态 | 建议 |
|------|----------|------|
| API Key | .env 文件 | ✅ 正确 |
| 密钥提交 | .gitignore 不包含 .env | ✅ 安全 |
| 日志泄露 | print() 可能泄露 | 使用 logging + 脱敏 |
| 输入验证 | 部分验证 | 统一验证 |

---

## 7.10 API 使用示例（完整流程）

```python
# 1. 导入
from helper_functions import encode_pdf, retrieve_context_per_question
from langchain_openai import ChatOpenAI
from langchain_core.prompts import PromptTemplate

# 2. 编码文档
vectorstore = encode_pdf("data/Understanding_Climate_Change.pdf")
retriever = vectorstore.as_retriever(search_kwargs={"k": 3})

# 3. 检索
context = retrieve_context_per_question("What is climate change?", retriever)

# 4. 生成答案
llm = ChatOpenAI(temperature=0, model_name="gpt-4o")
prompt = PromptTemplate(
    template="Answer based on context:\n{context}\nQuestion: {question}",
    input_variables=["context", "question"]
)
chain = prompt | llm
answer = chain.invoke({"context": "\n".join(context), "question": "What is climate change?"})

print(answer.content)
```

---

☕️ 制作不易，请我喝咖啡☕️关注我➕