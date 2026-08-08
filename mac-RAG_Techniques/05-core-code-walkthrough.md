# 5. 核心代码讲解 (Core Code Walkthrough)

> 本章对项目所有核心文件进行逐函数、逐类的深度走读，分析功能、参数、返回值、核心逻辑、设计模式、性能瓶颈与改进建议。预计字数：~20000 字。

---

## 5.1 核心文件重要性排序

| 优先级 | 文件 | 行数 | 重要性说明 |
|--------|------|------|-----------|
| **P0** | `helper_functions.py` | 362 | 全局共享核心，所有技术依赖 |
| **P0** | `evaluation/evalute_rag.py` | 155 | 评估框架核心 |
| **P1** | `simple_rag.py` | 115 | 基础 RAG 模板 |
| **P1** | `adaptive_retrieval.py` | 210 | 策略模式典范 |
| **P1** | `self_rag.py` | 200 | 自反思模式 |
| **P1** | `crag.py` | 250 | 纠错模式 |
| **P1** | `fusion_retrieval.py` | 145 | 混合检索 |
| **P2** | `reranking.py` | 180 | 重排序模式 |
| **P2** | `graph_rag.py` | 850+ | 知识图谱 |
| **P2** | `raptor.py` | 230 | 递归树 |
| **P2** | `hierarchical_indices.py` | 150 | 异步层次化 |
| **P3** | `semantic_chunking.py` | 140 | 语义切分 |
| **P3** | `contextual_compression.py` | 120 | 上下文压缩 |
| **P3** | `query_transformations.py` | 140 | 查询变换 |
| **P3** | `document_augmentation.py` | 185 | 文档增强 |
| **P3** | `context_enrichment_window_around_chunk.py` | 200 | 上下文扩展 |
| **P3** | `explainable_retrieval.py` | 90 | 可解释性 |
| **P3** | `retrieval_with_feedback_loop.py` | 150 | 反馈循环 |
| **P3** | `HyDe_Hypothetical_Document_Embedding.py` | 85 | HyDE |
| **P3** | `HyPE_Hypothetical_Prompt_Embeddings.py` | 210 | HyPE |
| **P3** | `choose_chunk_size.py` | 155 | 分块大小实验 |

---

## 5.2 P0: helper_functions.py 逐函数走读

### 5.2.1 文件概览

```python
"""
helper_functions.py - RAG Techniques 共享工具模块
作者: NirDiamant
行数: 362
职责: 提供所有 RAG 技术共享的编码、检索、问答、重试、工厂函数
"""
```

**导入依赖**：
```python
from langchain_community.document_loaders import PyPDFLoader
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain_openai import OpenAIEmbeddings
from langchain_community.vectorstores import FAISS
from pydantic import BaseModel, Field
from langchain_core.prompts import PromptTemplate
from openai import RateLimitError
from typing import List
from rank_bm25 import BM25Okapi
import fitz  # PyMuPDF
import asyncio
import random
import textwrap
import numpy as np
from enum import Enum
```

### 5.2.2 replace_t_with_space()

```python
def replace_t_with_space(list_of_documents):
    """
    Replaces all tab characters ('\t') with spaces in the page content 
    of each document.

    Args:
        list_of_documents: A list of document objects, each with a 
                          'page_content' attribute.

    Returns:
        The modified list of documents with tab characters replaced by spaces.
    """
    for doc in list_of_documents:
        doc.page_content = doc.page_content.replace('\t', ' ')
    return list_of_documents
```

**功能**：将文档列表中所有制表符替换为空格。  
**参数**：`list_of_documents` — LangChain Document 对象列表。  
**返回值**：修改后的文档列表（原地修改）。  
**设计考量**：
- 制表符会影响文本分割和嵌入质量
- 原地修改避免创建新对象（节省内存）
- 直接操作 `page_content` 属性

**潜在问题**：
- 无返回值检查（假设输入有效）
- 无类型检查（假设所有元素有 `page_content`）

**改进建议**：
```python
def replace_t_with_space(list_of_documents: List[Document]) -> List[Document]:
    if not list_of_documents:
        return []
    for doc in list_of_documents:
        if hasattr(doc, 'page_content') and isinstance(doc.page_content, str):
            doc.page_content = doc.page_content.replace('\t', ' ')
    return list_of_documents
```

### 5.2.3 encode_pdf()

```python
def encode_pdf(path, chunk_size=1000, chunk_overlap=200):
    """
    Encodes a PDF book into a vector store using OpenAI embeddings.

    Args:
        path: The path to the PDF file.
        chunk_size: The desired size of each text chunk.
        chunk_overlap: The amount of overlap between consecutive chunks.

    Returns:
        A FAISS vector store containing the encoded book content.
    """
    # 1. Load PDF documents
    loader = PyPDFLoader(path)
    documents = loader.load()

    # 2. Split documents into chunks
    text_splitter = RecursiveCharacterTextSplitter(
        chunk_size=chunk_size,
        chunk_overlap=chunk_overlap,
        length_function=len
    )
    texts = text_splitter.split_documents(documents)
    cleaned_texts = replace_t_with_space(texts)

    # 3. Create embeddings and vector store
    embeddings = OpenAIEmbeddings()
    vectorstore = FAISS.from_documents(cleaned_texts, embeddings)

    return vectorstore
```

**功能**：将 PDF 文档编码为 FAISS 向量存储。  
**参数**：
- `path` (str): PDF 文件路径
- `chunk_size` (int): 每个文本块的字符数（默认 1000）
- `chunk_overlap` (int): 相邻块的重叠字符数（默认 200）

**返回值**：`FAISS` 向量存储对象。

**核心逻辑**：
1. **加载**：`PyPDFLoader` 按页加载 PDF
2. **分块**：`RecursiveCharacterTextSplitter` 递归切分
3. **清洗**：`replace_t_with_space()` 替换制表符
4. **嵌入**：`OpenAIEmbeddings()` 生成向量
5. **存储**：`FAISS.from_documents()` 创建索引

**设计模式**：**Facade 模式** — 将多步复杂操作封装为单一接口。

**性能分析**：
- **时间复杂度**：O(n)，n 为 PDF 总字符数
- **空间复杂度**：O(n)，存储所有向量
- **API 调用**：`len(texts)` 次嵌入调用（可优化为批量）

**潜在问题**：
- 无路径验证（文件不存在会抛异常）
- 无 PDF 加密处理
- 无进度反馈（大 PDF 耗时较长）

**改进建议**：
```python
def encode_pdf(path, chunk_size=1000, chunk_overlap=200):
    if not os.path.exists(path):
        raise FileNotFoundError(f"PDF not found: {path}")
    # ... 原有逻辑 ...
```

### 5.2.4 encode_from_string()

```python
def encode_from_string(content, chunk_size=1000, chunk_overlap=200):
    """
    Encodes a string into a vector store using OpenAI embeddings.
    """
    # 输入验证
    if not isinstance(content, str) or not content.strip():
        raise ValueError("Content must be a non-empty string.")
    if not isinstance(chunk_size, int) or chunk_size <= 0:
        raise ValueError("chunk_size must be a positive integer.")
    if not isinstance(chunk_overlap, int) or chunk_overlap < 0:
        raise ValueError("chunk_overlap must be a non-negative integer.")

    try:
        text_splitter = RecursiveCharacterTextSplitter(
            chunk_size=chunk_size,
            chunk_overlap=chunk_overlap,
            length_function=len,
            is_separator_regex=False,
        )
        chunks = text_splitter.create_documents([content])
        for chunk in chunks:
            chunk.metadata['relevance_score'] = 1.0
        embeddings = OpenAIEmbeddings()
        vectorstore = FAISS.from_documents(chunks, embeddings)
    except Exception as e:
        raise RuntimeError(f"An error occurred during encoding: {str(e)}")
    return vectorstore
```

**功能**：将字符串编码为向量存储（相比 `encode_pdf` 增加了输入验证和错误处理）。  
**与 `encode_pdf` 的区别**：
- 输入为字符串而非文件路径
- 有完整的参数验证
- 有 try-except 错误处理
- 为每个 chunk 添加 `relevance_score` 元数据

**设计考量**：
- **防御式编程**：严格验证所有输入
- **错误传播**：捕获并重新抛出带上下文信息的异常
- **元数据扩展**：预留 `relevance_score` 字段供后续使用

### 5.2.5 retrieve_context_per_question()

```python
def retrieve_context_per_question(question, chunks_query_retriever):
    """
    Retrieves relevant context and unique URLs for a given question.
    """
    docs = chunks_query_retriever.get_relevant_documents(question)
    context = [doc.page_content for doc in docs]
    return context
```

**功能**：根据问题检索相关上下文。  
**参数**：
- `question` (str): 用户查询
- `chunks_query_retriever` (BaseRetriever): LangChain 检索器

**返回值**：`List[str]` — 上下文文本列表。

**设计考量**：
- 简单封装，统一接口
- 仅提取 `page_content`，丢弃元数据（简化下游处理）

### 5.2.6 create_question_answer_from_context_chain()

```python
class QuestionAnswerFromContext(BaseModel):
    """Model to generate an answer to a query based on a given context."""
    answer_based_on_content: str = Field(
        description="Generates an answer to a query based on a given context."
    )

def create_question_answer_from_context_chain(llm):
    """Creates a chain for answering questions from context."""
    question_answer_prompt_template = """ 
    For the question below, provide a concise but suffice answer 
    based ONLY on the provided context:
    {context}
    Question
    {question}
    """
    prompt = PromptTemplate(
        template=question_answer_prompt_template,
        input_variables=["context", "question"],
    )
    chain = prompt | llm.with_structured_output(QuestionAnswerFromContext)
    return chain
```

**功能**：创建基于上下文问答的 LLM 链。  
**设计模式**：**Chain 模式** — PromptTemplate + LLM + StructuredOutput。  
**关键技术**：`with_structured_output()` 确保输出符合 Pydantic Schema。

**Prompt 设计分析**：
- "based ONLY on the provided context" — 减少幻觉
- "concise but suffice" — 平衡简洁与完整

### 5.2.7 bm25_retrieval()

```python
def bm25_retrieval(bm25: BM25Okapi, cleaned_texts: List[str], 
                   query: str, k: int = 5) -> List[str]:
    """Perform BM25 retrieval and return the top k cleaned text chunks."""
    query_tokens = query.split()
    bm25_scores = bm25.get_scores(query_tokens)
    top_k_indices = np.argsort(bm25_scores)[::-1][:k]
    return [cleaned_texts[i] for i in top_k_indices]
```

**功能**：执行 BM25 关键词检索。  
**参数**：
- `bm25` (BM25Okapi): 预计算的 BM25 索引
- `cleaned_texts` (List[str]): 文本块列表
- `query` (str): 查询字符串
- `k` (int): 返回文档数

**返回值**：Top-k 文本块列表。

**算法复杂度**：
- 分词：O(m)，m 为查询长度
- 评分：O(n)，n 为文档总数
- 排序：O(n log n)

### 5.2.8 retry_with_exponential_backoff()

```python
async def retry_with_exponential_backoff(coroutine, max_retries=5):
    """Retries a coroutine using exponential backoff upon RateLimitError."""
    for attempt in range(max_retries):
        try:
            return await coroutine
        except RateLimitError as e:
            if attempt == max_retries - 1:
                raise e
            await exponential_backoff(attempt)
    raise Exception("Max retries reached")

async def exponential_backoff(attempt):
    """Implements exponential backoff with jitter."""
    wait_time = (2 ** attempt) + random.uniform(0, 1)
    print(f"Rate limit hit. Retrying in {wait_time:.2f} seconds...")
    await asyncio.sleep(wait_time)
```

**功能**：异步重试机制，处理 OpenAI API 速率限制。  
**设计模式**：**Retry 模式** + **指数退避** + **抖动**。  
**退避公式**：`wait = 2^attempt + random(0, 1)`  
**最大重试**：默认 5 次

**重试时间序列**：
| 尝试 | 等待时间（秒） |
|------|---------------|
| 0 | 1.0-2.0 |
| 1 | 2.0-3.0 |
| 2 | 4.0-5.0 |
| 3 | 8.0-9.0 |
| 4 | 16.0-17.0 |

**设计考量**：
- 仅捕获 `RateLimitError`（其他异常直接抛出）
- 抖动避免 **Thundering Herd** 问题
- 异步实现不阻塞事件循环

### 5.2.9 EmbeddingProvider & ModelProvider 枚举

```python
class EmbeddingProvider(Enum):
    OPENAI = "openai"
    COHERE = "cohere"
    AMAZON_BEDROCK = "bedrock"

class ModelProvider(Enum):
    OPENAI = "openai"
    GROQ = "groq"
    ANTHROPIC = "anthropic"
    AMAZON_BEDROCK = "bedrock"

def get_langchain_embedding_provider(provider: EmbeddingProvider, 
                                     model_id: str = None):
    """Returns an embedding provider based on the specified provider."""
    if provider == EmbeddingProvider.OPENAI:
        return OpenAIEmbeddings()
    elif provider == EmbeddingProvider.COHERE:
        return CohereEmbeddings()
    elif provider == EmbeddingProvider.AMAZON_BEDROCK:
        return BedrockEmbeddings(model_id=model_id) if model_id else \
               BedrockEmbeddings(model_id="amazon.titan-embed-text-v2:0")
    else:
        raise ValueError(f"Unsupported embedding provider: {provider}")
```

**功能**：多 Provider 支持的工厂函数。  
**设计模式**：**Strategy 模式** + **Factory 模式**。  
**扩展性**：新增 Provider 只需添加枚举值和对应分支。

---

## 5.3 P0: evaluation/evalute_rag.py 逐函数走读

### 5.3.1 文件概览

```python
"""
evalute_rag.py - RAG 评估框架
集成 DeepEval 库，提供多维度 RAG 质量评估
"""
```

### 5.3.2 create_deep_eval_test_cases()

```python
def create_deep_eval_test_cases(
    questions: List[str],
    gt_answers: List[str],
    generated_answers: List[str],
    retrieved_documents: List[str]
) -> List[LLMTestCase]:
    """Create LLMTestCase objects for evaluation."""
    return [
        LLMTestCase(
            input=question,
            expected_output=gt_answer,
            actual_output=generated_answer,
            retrieval_context=retrieved_document
        )
        for question, gt_answer, generated_answer, retrieved_document 
        in zip(questions, gt_answers, generated_answers, retrieved_documents)
    ]
```

**功能**：创建 DeepEval 评估测试用例。  
**参数**：问题列表、标准答案列表、生成答案列表、检索文档列表。  
**返回值**：`LLMTestCase` 对象列表。

### 5.3.3 评估指标定义

```python
# Correctness (GEval)
correctness_metric = GEval(
    name="Correctness",
    model="gpt-4-turbo",
    evaluation_params=[
        LLMTestCaseParams.EXPECTED_OUTPUT,
        LLMTestCaseParams.ACTUAL_OUTPUT
    ],
    evaluation_steps=[
        "Determine whether the actual output is factually correct "
        "based on the expected output."
    ],
)

# Faithfulness
faithfulness_metric = FaithfulnessMetric(
    threshold=0.7,
    model="gpt-4-turbo",
    include_reason=False
)

# Contextual Relevancy
relevance_metric = ContextualRelevancyMetric(
    threshold=1,
    model="gpt-4-turbo",
    include_reason=True
)
```

**指标说明**：

| 指标 | 类型 | 阈值 | Judge 模型 | 说明 |
|------|------|------|-----------|------|
| Correctness | Generation | - | GPT-4-turbo | 答案事实正确性 |
| Faithfulness | Generation | 0.7 | GPT-4-turbo | 答案基于上下文程度 |
| ContextualRelevancy | Retrieval | 1 | GPT-4-turbo | 检索结果相关性 |

### 5.3.4 evaluate_rag()

```python
def evaluate_rag(retriever, num_questions: int = 5) -> Dict[str, Any]:
    """Evaluates a RAG system using predefined test questions."""
    llm = ChatOpenAI(temperature=0, model_name="gpt-4-turbo")
    
    eval_prompt = PromptTemplate.from_template("""
    Evaluate the following retrieval results for the question.
    Question: {question}
    Retrieved Context: {context}
    Rate on a scale of 1-5 (5 being best) for:
    1. Relevance  2. Completeness  3. Conciseness
    Provide ratings in JSON format:
    """)
    
    eval_chain = eval_prompt | llm | StrOutputParser()
    
    # 生成测试问题
    question_chain = question_gen_prompt | llm | StrOutputParser()
    questions = question_chain.invoke(...).split("\n")
    
    results = []
    for question in questions:
        context = retriever.get_relevant_documents(question)
        eval_result = eval_chain.invoke({"question": question, "context": context})
        results.append(eval_result)
    
    return {"questions": questions, "results": results, 
            "average_scores": calculate_average_scores(results)}
```

**功能**：执行 RAG 系统评估。  
**流程**：
1. 生成/加载测试问题
2. 对每个问题执行检索
3. LLM 评估检索质量
4. 汇总计算平均分

**潜在问题**：
- `calculate_average_scores()` 未实现（pass）
- 评估 Prompt 要求 JSON 格式但无 Schema 校验

---

## 5.4 P1: simple_rag.py 逐函数走读

### 5.4.1 SimpleRAG 类

```python
class SimpleRAG:
    """基础 RAG 实现"""
    
    def __init__(self, path, chunk_size=1000, chunk_overlap=200, n_retrieved=2):
        """初始化：编码 PDF + 创建检索器"""
        self.vector_store = encode_pdf(path, chunk_size, chunk_overlap)
        self.chunks_query_retriever = self.vector_store.as_retriever(
            search_kwargs={"k": n_retrieved}
        )
    
    def run(self, query):
        """执行检索并展示结果"""
        context = retrieve_context_per_question(query, self.chunks_query_retriever)
        show_context(context)
```

**设计分析**：
- **单一职责**：仅负责检索，不负责生成
- **可配置**：所有参数可通过构造函数调整
- **计时**：记录 Chunking 和 Retrieval 时间

### 5.4.2 CLI 参数处理

```python
def parse_args():
    parser = argparse.ArgumentParser(description="Encode a PDF and test RAG.")
    parser.add_argument("--path", type=str, 
                       default="../data/Understanding_Climate_Change.pdf")
    parser.add_argument("--chunk_size", type=int, default=1000)
    parser.add_argument("--chunk_overlap", type=int, default=200)
    parser.add_argument("--n_retrieved", type=int, default=2)
    parser.add_argument("--query", type=str, 
                       default="What is the main cause of climate change?")
    parser.add_argument("--evaluate", action="store_true")
    return validate_args(parser.parse_args())
```

**设计分析**：
- 合理的默认值（Climate Change PDF）
- 参数验证函数 `validate_args()`
- `--evaluate` 标志启用评估

---

## 5.5 P1: adaptive_retrieval.py 逐函数走读

### 5.5.1 QueryClassifier 类

```python
class QueryClassifier:
    """查询分类器"""
    def __init__(self):
        self.llm = ChatOpenAI(temperature=0, model_name="gpt-4o", max_tokens=4000)
        self.prompt = PromptTemplate(
            input_variables=["query"],
            template="Classify the following query into one of these categories: "
                    "Factual, Analytical, Opinion, or Contextual.\n"
                    "Query: {query}\nCategory:"
        )
        self.chain = self.prompt | self.llm.with_structured_output(CategoriesOptions)
    
    def classify(self, query):
        return self.chain.invoke(query).category
```

**设计分析**：
- 使用 `with_structured_output()` 确保输出为预定义类别
- 温度 0 确保确定性输出

### 5.5.2 四种检索策略

```python
class BaseRetrievalStrategy:
    """检索策略基类"""
    def __init__(self, texts):
        self.embeddings = OpenAIEmbeddings()
        text_splitter = CharacterTextSplitter(chunk_size=800, chunk_overlap=0)
        self.documents = text_splitter.create_documents(texts)
        self.db = FAISS.from_documents(self.documents, self.embeddings)
        self.llm = ChatOpenAI(temperature=0, model_name="gpt-4o", max_tokens=4000)
    
    def retrieve(self, query, k=4):
        return self.db.similarity_search(query, k=k)

class FactualRetrievalStrategy(BaseRetrievalStrategy):
    """事实查询策略：增强查询 + 评分排序"""
    def retrieve(self, query, k=4):
        enhanced_query = self._enhance_query(query)  # LLM 增强
        docs = self.db.similarity_search(enhanced_query, k=k*2)  # 宽松检索
        ranked_docs = self._rank_documents(query, docs)  # LLM 评分
        return ranked_docs[:k]

class AnalyticalRetrievalStrategy(BaseRetrievalStrategy):
    """分析查询策略：子查询分解 + 多样性选择"""
    def retrieve(self, query, k=4):
        sub_queries = self._generate_sub_queries(query)  # 生成子查询
        all_docs = []
        for sub_query in sub_queries:
            all_docs.extend(self.db.similarity_search(sub_query, k=2))
        selected_indices = self._select_diverse_docs(query, all_docs, k)
        return [all_docs[i] for i in selected_indices]

class OpinionRetrievalStrategy(BaseRetrievalStrategy):
    """观点查询策略：多视角检索"""
    def retrieve(self, query, k=3):
        viewpoints = self._identify_viewpoints(query)  # 识别观点
        all_docs = []
        for viewpoint in viewpoints:
            all_docs.extend(self.db.similarity_search(f"{query} {viewpoint}", k=2))
        selected_indices = self._classify_opinions(query, all_docs, k)
        return [all_docs[int(i)] for i in selected_indices]

class ContextualRetrievalStrategy(BaseRetrievalStrategy):
    """上下文查询策略：用户上下文改写"""
    def retrieve(self, query, k=4, user_context=None):
        contextualized_query = self._contextualize_query(query, user_context)
        docs = self.db.similarity_search(contextualized_query, k=k*2)
        ranked_docs = self._rank_with_context(query, docs, user_context)
        return ranked_docs[:k]
```

**设计模式分析**：
- **Strategy 模式**：四种策略可互换
- **Template Method**：基类定义框架，子类实现细节
- **开闭原则**：新增策略无需修改现有代码

### 5.5.3 AdaptiveRAG 主类

```python
class AdaptiveRAG:
    """自适应 RAG 主类"""
    def __init__(self, texts: List[str]):
        self.classifier = QueryClassifier()
        self.strategies = {
            "Factual": FactualRetrievalStrategy(texts),
            "Analytical": AnalyticalRetrievalStrategy(texts),
            "Opinion": OpinionRetrievalStrategy(texts),
            "Contextual": ContextualRetrievalStrategy(texts)
        }
        self.llm = ChatOpenAI(temperature=0, model_name="gpt-4o", max_tokens=4000)
        self.prompt = PromptTemplate(
            template="Use the context to answer...\n{context}\nQuestion: {question}",
            input_variables=["context", "question"]
        )
        self.llm_chain = self.prompt | self.llm
    
    def answer(self, query: str) -> str:
        category = self.classifier.classify(query)
        strategy = self.strategies[category]
        docs = strategy.retrieve(query)
        input_data = {"context": "\n".join([d.page_content for d in docs]), 
                     "question": query}
        return self.llm_chain.invoke(input_data).content
```

**性能分析**：
- 分类：1 次 LLM 调用
- 策略执行：1-2 次 LLM 调用（视策略而定）
- 生成：1 次 LLM 调用
- **总计**：3-4 次 LLM 调用/查询

---

## 5.6 P1: self_rag.py 逐函数走读

### 5.6.1 结构化输出模型

```python
class RetrievalResponse(BaseModel):
    response: str = Field(..., description="Output only 'Yes' or 'No'.")

class RelevanceResponse(BaseModel):
    response: str = Field(..., description="Output 'Relevant' or 'Irrelevant'.")

class GenerationResponse(BaseModel):
    response: str = Field(..., description="The generated response.")

class SupportResponse(BaseModel):
    response: str = Field(..., description="Output 'Fully supported', "
                          "'Partially supported', or 'No support'.")

class UtilityResponse(BaseModel):
    response: int = Field(..., description="Rate the utility from 1 to 5.")
```

### 5.6.2 6 步流水线

```python
class SelfRAG:
    def run(self, query):
        # Step 1: 检索决策
        retrieval_decision = self.retrieval_chain.invoke({"query": query}).response
        
        if retrieval_decision == 'yes':
            # Step 2: 检索文档
            docs = self.vectorstore.similarity_search(query, k=self.top_k)
            
            # Step 3: 相关性评估
            relevant_contexts = []
            for context in contexts:
                relevance = self.relevance_chain.invoke(...)
                if relevance == 'relevant':
                    relevant_contexts.append(context)
            
            # Step 4-6: 生成 + 评估
            responses = []
            for context in relevant_contexts:
                response = self.generation_chain.invoke(...)
                support = self.support_chain.invoke(...)
                utility = self.utility_chain.invoke(...)
                responses.append((response, support, utility))
            
            # 选择最佳
            best = max(responses, key=lambda x: (x[1] == 'fully supported', x[2]))
            return best[0]
        else:
            return self.generation_chain.invoke(...).response
```

**性能分析**：
- 最坏情况：1 (决策) + k (相关性) + k (生成) + k (支持度) + k (效用) = 4k+1 次 LLM 调用
- 当 k=3 时，最多 13 次 LLM 调用
- **优化空间**：可并行执行 Step 4-6

---

## 5.7 P1: crag.py 逐函数走读

### 5.7.1 CRAG 类

```python
class CRAG:
    def __init__(self, path, model="gpt-4o-mini", max_tokens=1000, 
                 temperature=0, lower_threshold=0.3, upper_threshold=0.7):
        self.vectorstore = encode_pdf(path)
        self.llm = ChatOpenAI(model=model, max_tokens=max_tokens, temperature=temperature)
        self.search = DuckDuckGoSearchResults()
        self.lower_threshold = lower_threshold
        self.upper_threshold = upper_threshold
```

### 5.7.2 核心方法

```python
def evaluate_documents(self, query, documents):
    """评估文档相关性"""
    return [self.retrieval_evaluator(query, doc) for doc in documents]

def retrieval_evaluator(self, query, document):
    """单个文档评估"""
    prompt = PromptTemplate(
        template="On a scale from 0 to 1, how relevant is the following "
                "document to the query?\nQuery: {query}\nDocument: {document}\n"
                "Relevance score:"
    )
    chain = prompt | self.llm.with_structured_output(RetrievalEvaluatorInput)
    return chain.invoke({"query": query, "document": document}).relevance_score

def knowledge_refinement(self, document):
    """知识精炼：提取关键点"""
    prompt = PromptTemplate(
        template="Extract the key information from the following document "
                "in bullet points:\n{document}\nKey points:"
    )
    chain = prompt | self.llm.with_structured_output(KnowledgeRefinementInput)
    result = chain.invoke({"document": document}).key_points
    return [p.strip() for p in result.split('\n') if p.strip()]

def rewrite_query(self, query):
    """查询改写（用于网络搜索）"""
    prompt = PromptTemplate(
        template="Rewrite the following query to make it more suitable "
                "for a web search:\n{query}\nRewritten query:"
    )
    chain = prompt | self.llm.with_structured_output(QueryRewriterInput)
    return chain.invoke({"query": query}).query.strip()
```

**阈值逻辑**：
```python
if score > upper_threshold:      # > 0.7
    # 直接使用检索结果
elif score > lower_threshold:    # 0.3 - 0.7
    # 知识精炼 + 补充
else:                            # < 0.3
    # 查询改写 + 网络搜索
```

---

## 5.8 P2: graph_rag.py 逐函数走读

### 5.8.1 DocumentProcessor 类

```python
class DocumentProcessor:
    def __init__(self):
        self.text_splitter = RecursiveCharacterTextSplitter(
            chunk_size=1000, chunk_overlap=200
        )
        self.embeddings = OpenAIEmbeddings()
    
    def process_documents(self, documents):
        """处理文档：分块 + 向量存储"""
        splits = self.text_splitter.split_documents(documents)
        vector_store = FAISS.from_documents(splits, self.embeddings)
        return splits, vector_store
    
    def create_embeddings_batch(self, texts, batch_size=32):
        """批量嵌入"""
        embeddings = []
        for i in range(0, len(texts), batch_size):
            batch = texts[i:i + batch_size]
            batch_embeddings = self.embeddings.embed_documents(batch)
            embeddings.extend(batch_embeddings)
        return np.array(embeddings)
    
    def compute_similarity_matrix(self, embeddings):
        """余弦相似度矩阵"""
        return cosine_similarity(embeddings)
```

### 5.8.2 KnowledgeGraph 类

```python
class KnowledgeGraph:
    def __init__(self):
        self.graph = nx.Graph()
        self.lemmatizer = WordNetLemmatizer()
        self.concept_cache = {}
        self.nlp = self._load_spacy_model()
        self.edges_threshold = 0.8
    
    def build_graph(self, splits, llm, embedding_model):
        """构建知识图谱"""
        self._add_nodes(splits)
        embeddings = self._create_embeddings(splits, embedding_model)
        self._extract_concepts(splits, llm)
        self._add_edges(embeddings)
    
    def _add_nodes(self, splits):
        """添加节点"""
        for i, split in enumerate(splits):
            self.graph.add_node(i, text=split.page_content, 
                               metadata=split.metadata)
    
    def _extract_concepts(self, splits, llm):
        """概念提取：spaCy NER + LLM"""
        for i, split in enumerate(splits):
            # spaCy NER
            doc = self.nlp(split.page_content)
            named_entities = [ent.text for ent in doc.ents]
            
            # LLM 概念提取
            prompt = PromptTemplate(
                template="Extract key concepts (excluding named entities) "
                        "from the following text:\n\n{text}\n\nKey concepts:"
            )
            chain = prompt | llm.with_structured_output(Concepts)
            llm_concepts = chain.invoke({"text": split.page_content}).concepts_list
            
            # 合并去重
            all_concepts = list(set(named_entities + llm_concepts))
            self.graph.nodes[i]['concepts'] = all_concepts
    
    def _add_edges(self, embeddings):
        """基于相似度添加边"""
        similarity_matrix = cosine_similarity(embeddings)
        for i in range(len(embeddings)):
            for j in range(i + 1, len(embeddings)):
                if similarity_matrix[i][j] > self.edges_threshold:
                    self.graph.add_edge(i, j, 
                                       weight=similarity_matrix[i][j])
```

**复杂度分析**：
- 节点添加：O(n)
- 概念提取：O(n) × LLM 调用
- 边添加：O(n²)（相似度矩阵）

---

## 5.9 P2: fusion_retrieval.py 逐函数走读

### 5.9.1 核心融合函数

```python
def fusion_retrieval(vectorstore, bm25, query: str, k: int = 5, 
                     alpha: float = 0.5) -> List[Document]:
    """混合检索：向量 + BM25"""
    # 获取所有文档
    all_docs = vectorstore.similarity_search("", k=vectorstore.index.ntotal)
    
    # BM25 检索
    bm25_scores = bm25.get_scores(query.split())
    
    # 向量检索
    vector_results = vectorstore.similarity_search_with_score(query, k=len(all_docs))
    vector_scores = np.array([score for _, score in vector_results])
    
    # 归一化
    vector_scores = 1 - (vector_scores - np.min(vector_scores)) / \
                        (np.max(vector_scores) - np.min(vector_scores))
    bm25_scores = (bm25_scores - np.min(bm25_scores)) / \
                  (np.max(bm25_scores) - np.min(bm25_scores))
    
    # 加权融合
    combined_scores = alpha * vector_scores + (1 - alpha) * bm25_scores
    sorted_indices = np.argsort(combined_scores)[::-1]
    
    return [all_docs[i] for i in sorted_indices[:k]]
```

**算法分析**：
- **时间复杂度**：O(n log n)（排序主导）
- **空间复杂度**：O(n)（存储所有分数）
- **参数 alpha**：0.5 等权，>0.5 偏向向量，<0.5 偏向 BM25

---

## 5.10 P2: reranking.py 逐函数走读

### 5.10.1 LLM 评分重排序

```python
def rerank_documents(query: str, docs: List[Document], top_n: int = 3) -> List[Document]:
    """LLM 评分重排序"""
    prompt_template = PromptTemplate(
        input_variables=["query", "doc"],
        template="On a scale of 1-10, rate the relevance of the following "
                "document to the query. Consider the specific context and "
                "intent of the query, not just keyword matches.\n"
                "Query: {query}\nDocument: {doc}\nRelevance Score:"
    )
    llm = ChatOpenAI(temperature=0, model_name="gpt-4o", max_tokens=4000)
    llm_chain = prompt_template | llm.with_structured_output(RatingScore)
    
    scored_docs = []
    for doc in docs:
        score = llm_chain.invoke({"query": query, "doc": doc.page_content}).relevance_score
        scored_docs.append((doc, float(score)))
    
    reranked_docs = sorted(scored_docs, key=lambda x: x[1], reverse=True)
    return [doc for doc, _ in reranked_docs[:top_n]]
```

### 5.10.2 Cross-Encoder 重排序

```python
class CrossEncoderRetriever(BaseRetriever, BaseModel):
    vectorstore: Any
    cross_encoder: Any
    k: int = 5
    rerank_top_k: int = 3
    
    def get_relevant_documents(self, query: str) -> List[Document]:
        initial_docs = self.vectorstore.similarity_search(query, k=self.k)
        pairs = [[query, doc.page_content] for doc in initial_docs]
        scores = self.cross_encoder.predict(pairs)
        scored_docs = sorted(zip(initial_docs, scores), key=lambda x: x[1], reverse=True)
        return [doc for doc, _ in scored_docs[:self.rerank_top_k]]
```

**对比**：

| 方法 | 速度 | 精度 | 成本 |
|------|------|------|------|
| LLM 评分 | 慢（API 调用） | 高 | 高 |
| Cross-Encoder | 快（本地推理） | 中高 | 低 |

---

## 5.11 P3: 其他技术简要分析

### 5.11.1 semantic_chunking.py

```python
class SemanticChunkingRAG:
    def __init__(self, path, breakpoint_type="percentile", breakpoint_amount=90):
        content = read_pdf_to_string(path)
        self.semantic_chunker = SemanticChunker(
            self.embeddings,
            breakpoint_threshold_type=breakpoint_type,
            breakpoint_threshold_amount=breakpoint_amount
        )
        self.semantic_docs = self.semantic_chunker.create_documents([content])
        self.semantic_vectorstore = FAISS.from_documents(self.semantic_docs, self.embeddings)
```

**核心**：使用 `SemanticChunker`（基于嵌入距离的断点检测）替代固定大小切分。

### 5.11.2 contextual_compression.py

```python
class ContextualCompressionRAG:
    def __init__(self, path, model_name="gpt-4o-mini"):
        self.vector_store = encode_pdf(path)
        self.retriever = self.vector_store.as_retriever()
        self.llm = ChatOpenAI(temperature=0, model_name=model_name, max_tokens=4000)
        self.compressor = LLMChainExtractor.from_llm(self.llm)
        self.compression_retriever = ContextualCompressionRetriever(
            base_compressor=self.compressor,
            base_retriever=self.retriever
        )
        self.qa_chain = RetrievalQA.from_chain_type(
            llm=self.llm, retriever=self.compression_retriever,
            return_source_documents=True
        )
```

**核心**：`LLMChainExtractor` 使用 LLM 从检索结果中提取与查询相关的部分。

### 5.11.3 query_transformations.py

```python
class RAGQueryProcessor:
    def __init__(self):
        # 三种查询变换
        self.query_rewriter = PromptTemplate(...) | self.re_write_llm        # 重写
        self.step_back_chain = PromptTemplate(...) | self.step_back_llm      # Step-back
        self.subquery_decomposer_chain = PromptTemplate(...) | self.sub_query_llm  # 分解
```

**核心**：三种查询变换技术：
1. **Query Rewriting**：改写为更具体
2. **Step-back Prompting**：生成更宽泛查询
3. **Sub-query Decomposition**：分解为子查询

### 5.11.4 document_augmentation.py

```python
def generate_questions(text: str) -> List[str]:
    """为文档生成可能的问题"""
    llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
    prompt = PromptTemplate(
        template="Using the context data: {context}\n\nGenerate a list of "
                "at least {num_questions} possible questions..."
    )
    chain = prompt | llm.with_structured_output(QuestionList)
    result = chain.invoke({"context": text, "num_questions": QUESTIONS_PER_DOCUMENT})
    return clean_and_filter_questions(result.question_list)
```

**核心**：为每个文档生成可能的问题，增强检索覆盖率。

### 5.11.5 hierarchical_indices.py

```python
async def encode_pdf_hierarchical(path, chunk_size=1000, chunk_overlap=200):
    """异步层次化编码"""
    # 1. 文档级摘要（map_reduce）
    summary_chain = load_summarize_chain(summary_llm, chain_type="map_reduce")
    summaries = []
    for batch in batches:
        batch_summaries = await asyncio.gather(*[summarize_doc(doc) for doc in batch])
        summaries.extend(batch_summaries)
    
    # 2. 详细块切分
    detailed_chunks = text_splitter.split_documents(documents)
    
    # 3. 创建两个向量存储
    summary_vectorstore, detailed_vectorstore = await asyncio.gather(
        create_vectorstore(summaries),
        create_vectorstore(detailed_chunks)
    )
    return summary_vectorstore, detailed_vectorstore
```

**核心**：双层索引 + 异步处理 + 指数退避重试。

---

## 5.12 代码质量评估

### 5.12.1 优点

| 维度 | 评分 | 说明 |
|------|------|------|
| **一致性** | ★★★★★ | 统一的函数签名、类结构、CLI 模式 |
| **可读性** | ★★★★☆ | 清晰的命名、文档字符串、类型提示 |
| **错误处理** | ★★★☆☆ | 部分有验证，但不够统一 |
| **可测试性** | ★★★☆☆ | 有测试框架，但测试覆盖有限 |
| **可扩展性** | ★★★★★ | 新增技术只需添加文件 |
| **性能** | ★★★☆☆ | 无缓存、无批处理优化 |

### 5.12.2 潜在问题

1. **硬编码**：部分脚本硬编码了默认路径和模型名称
2. **无日志**：使用 `print()` 而非 `logging` 模块
3. **无缓存**：每次运行重新编码 PDF
4. **无批处理**：嵌入调用未充分利用批处理
5. **异常处理不统一**：部分函数有 try-except，部分没有

### 5.12.3 改进建议

1. **引入日志**：
```python
import logging
logger = logging.getLogger(__name__)
# 替代 print()
logger.info("Encoding PDF...")
```

2. **添加缓存**：
```python
import hashlib
def get_cache_key(path, chunk_size, chunk_overlap):
    content = open(path, 'rb').read()
    return hashlib.md5(f"{content}{chunk_size}{chunk_overlap}".encode()).hexdigest()
```

3. **统一错误处理**：
```python
def safe_encode(path, **kwargs):
    try:
        return encode_pdf(path, **kwargs)
    except FileNotFoundError:
        logger.error(f"File not found: {path}")
        raise
    except Exception as e:
        logger.error(f"Encoding failed: {e}")
        raise RuntimeError(f"Failed to encode {path}") from e
```

4. **配置外部化**：
```python
# config.yaml
default:
  model: gpt-4o-mini
  chunk_size: 1000
  chunk_overlap: 200
  embedding_model: text-embedding-3
```

---

# ❤️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../res/promotion.jpg)