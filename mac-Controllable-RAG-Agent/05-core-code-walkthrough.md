# 5. 核心代码讲解 (Core Code Walkthrough)

> **章节编号**: 05/13  
> **预计篇幅**: ~20,000 字  
> **覆盖范围**: 所有核心文件逐函数深度走读  
> **关联文件**: helper_functions.py, functions_for_pipeline.py, simulate_agent.py

---

## 5.1 代码走读总览

### 5.1.1 文件重要度排序

| 优先级 | 文件 | 行数 | 重要度 | 原因 |
|--------|------|------|--------|------|
| ⭐⭐⭐⭐⭐ | `functions_for_pipeline.py` | 1182 | 最高 | 系统核心，包含所有 Chain 和图编排 |
| ⭐⭐⭐⭐ | `simulate_agent.py` | 238 | 高 | 唯一用户交互入口 |
| ⭐⭐⭐ | `helper_functions.py` | 245 | 中 | 基础工具，被核心模块依赖 |
| ⭐⭐ | Notebook | - | 低 | 教程性质，非生产代码 |

### 5.1.2 走读方法论

每个函数按以下维度分析：
- **功能**: 做什么
- **参数**: 输入什么
- **返回值**: 输出什么
- **核心逻辑**: 怎么实现
- **设计模式**: 使用了什么模式
- **潜在问题**: 有什么缺陷
- **改进建议**: 如何优化

---

## 5.2 `helper_functions.py` 逐函数走读

### 5.2.1 `num_tokens_from_string(string: str, encoding_name: str) -> int`

```python
def num_tokens_from_string(string: str, encoding_name: str) -> int:
    encoding = tiktoken.encoding_for_model(encoding_name)
    num_tokens = len(encoding.encode(string))
    return num_tokens
```

| 维度 | 分析 |
|------|------|
| **功能** | 计算字符串在指定模型编码下的 token 数 |
| **参数** | `string`: 待计算文本；`encoding_name`: 模型名称（如 "gpt-3.5-turbo-0125"） |
| **返回值** | token 数量（int） |
| **核心逻辑** | 调用 tiktoken 的 `encoding_for_model()` 获取编码器，然后 `encode()` 编码并计数 |
| **设计模式** | 纯函数，无副作用 |
| **潜在问题** | `encoding_name` 无效时会抛出 `KeyError` |
| **改进建议** | 添加 try-except 处理无效模型名 |

---

### 5.2.2 `split_into_chapters(book_path: str) -> list`

```python
def split_into_chapters(book_path):
    with open(book_path, 'rb') as pdf_file:
        pdf_reader = PyPDF2.PdfReader(pdf_file)
        documents = pdf_reader.pages
        text = " ".join([doc.extract_text() for doc in documents])
        chapters = re.split(r'(CHAPTER\s[A-Z]+(?:\s[A-Z]+)*)', text)
        
        chapter_docs = []
        chapter_num = 1
        for i in range(1, len(chapters), 2):
            chapter_text = chapters[i] + chapters[i + 1]
            doc = Document(page_content=chapter_text, metadata={"chapter": chapter_num})
            chapter_docs.append(doc)
            chapter_num += 1
    return chapter_docs
```

| 维度 | 分析 |
|------|------|
| **功能** | 将 PDF 书籍按章节标题分割为 Document 列表 |
| **参数** | `book_path`: PDF 文件路径 |
| **返回值** | `List[Document]`，每个 Document 包含章节文本和章节号元数据 |
| **核心逻辑** | 1. PyPDF2 读取全部页 → 2. 拼接全文 → 3. 正则分割 → 4. 构造 Document |
| **设计模式** | 流水线模式（加载→提取→分割→封装） |
| **潜在问题** | ① 正则假设章节标题全大写；② 前言/目录会被错误合并；③ 无错误处理 |
| **改进建议** | ① 添加文件存在检查；② 支持自定义正则；③ 过滤空章节 |

**正则详解**:
```
r'(CHAPTER\s[A-Z]+(?:\s[A-Z]+)*)'
├── CHAPTER     : 匹配字面量 "CHAPTER"
├── \s          : 匹配空格
├── [A-Z]+      : 匹配一个大写字母（如 "ONE"）
└── (?:\s[A-Z]+)* : 可选的额外大写词（如 "THE BOY WHO LIVED"）
```

---

### 5.2.3 `extract_book_quotes_as_documents(documents, min_length=50)`

```python
def extract_book_quotes_as_documents(documents, min_length=50):
    quotes_as_documents = []
    quote_pattern_longer_than_min_length = re.compile(rf'“(.{{{min_length},}}?)”', re.DOTALL)
    
    for doc in documents:
        content = doc.page_content
        content = content.replace('\n', ' ')
        found_quotes = quote_pattern_longer_than_min_length.findall(content)
        for quote in found_quotes:
            quote_doc = Document(page_content=quote)
            quotes_as_documents.append(quote_doc)
    return quotes_as_documents
```

| 维度 | 分析 |
|------|------|
| **功能** | 从文档中提取引号内容（≥50 字符）作为独立 Document |
| **参数** | `documents`: Document 列表；`min_length`: 最小引号长度 |
| **返回值** | `List[Document]`，每个 Document 是一条引文 |
| **核心逻辑** | 1. 编译正则 → 2. 遍历文档 → 3. 替换换行 → 4. 正则匹配 → 5. 封装 Document |
| **潜在问题** | ① 使用中文引号 `"` `"`，英文书可能不匹配；② `min_length` 硬编码 |
| **改进建议** | 同时支持英文引号 `"` `'`；可配置 min_length |

---

### 5.2.4 `escape_quotes(text: str) -> str`

```python
def escape_quotes(text):
    return text.replace('"', '\\"').replace("'", "\\'")
```

| 维度 | 分析 |
|------|------|
| **功能** | 转义单引号和双引号 |
| **参数** | `text`: 原始文本 |
| **返回值** | 转义后的文本 |
| **核心逻辑** | 链式调用 `replace()` |
| **潜在问题** | ① 如果文本已转义，会双重转义；② 不处理反斜杠本身 |
| **改进建议** | 使用 `json.dumps()` 或 `repr()` 替代手动转义 |

---

### 5.2.5 `is_similarity_ratio_lower_than_th(large_string, short_string, th) -> bool`

```python
def is_similarity_ratio_lower_than_th(large_string, short_string, th):
    lcs = pylcs.lcs_sequence_length(large_string, short_string)
    similarity_ratio = lcs / len(short_string)
    if similarity_ratio < th:
        return True
    else:
        return False
```

| 维度 | 分析 |
|------|------|
| **功能** | 判断长字符串与短字符串的 LCS 相似度是否低于阈值 |
| **参数** | `large_string`: 长文本；`short_string`: 短文本；`th`: 阈值 |
| **返回值** | `True` 表示相似度低于阈值（不相似） |
| **核心逻辑** | 计算最长公共子序列（LCS），除以短字符串长度得到比例 |
| **潜在问题** | ① `short_string` 为空时会 `ZeroDivisionError`；② LCS 时间复杂度 O(n×m) |
| **改进建议** | 添加空字符串检查；考虑使用编辑距离替代 |

**LCS 算法复杂度**:
- **时间**: O(n × m)，n 和 m 为两字符串长度
- **空间**: O(min(n, m))（pylcs 使用优化版本）

---

### 5.2.6 `analyse_metric_results(results_df: pd.DataFrame) -> None`

```python
def analyse_metric_results(results_df):
    for metric_name, metric_value in results_df.items():
        print(f"\n**{metric_name.upper()}**")
        if isinstance(metric_value, pd.Series):
            metric_value = metric_value.values[0]
        
        if metric_name == "faithfulness":
            print("Measures how well the generated answer is supported by the retrieved documents.")
            print(f"Score: {metric_value:.4f}")
        # ... 其他指标类似
```

| 维度 | 分析 |
|------|------|
| **功能** | 打印 Ragas 评估指标的解释和分数 |
| **参数** | `results_df`: Ragas evaluate() 返回的 DataFrame |
| **返回值** | None（仅打印） |
| **核心逻辑** | 遍历 DataFrame 的每个指标，打印名称、解释、分数 |
| **潜在问题** | ① 硬编码指标名称；② 不返回结构化结果 |
| **改进建议** | 使用字典映射指标名称到解释；返回结构化结果 |

---

### 5.2.7 `save_object(obj, filename)` / `load_object(filename)`

```python
def save_object(obj, filename):
    with open(filename, 'wb') as file:
        dill.dump(obj, file)
    print(f"Object has been saved to '{filename}'.")

def load_object(filename):
    with open(filename, 'rb') as file:
        obj = dill.load(file)
    print(f"Object has been loaded from '{filename}'.")
    return obj
```

| 维度 | 分析 |
|------|------|
| **功能** | 使用 dill 序列化/反序列化 Python 对象 |
| **参数** | `obj`: 待保存对象；`filename`: 文件路径 |
| **核心逻辑** | dill 是 pickle 的增强版，支持 lambda、闭包等 |
| **潜在问题** | ① 反序列化不可信文件有安全风险；② 版本兼容性 |
| **改进建议** | 添加文件校验（hash）；使用版本标记 |

---

## 5.3 `functions_for_pipeline.py` 逐函数走读

### 5.3.1 模块初始化部分

#### `create_retrievers() -> tuple`

```python
def create_retrievers():
    embeddings = OpenAIEmbeddings()
    chunks_vector_store = FAISS.load_local("chunks_vector_store", embeddings, allow_dangerous_deserialization=True)
    chapter_summaries_vector_store = FAISS.load_local("chapter_summaries_vector_store", embeddings, allow_dangerous_deserialization=True)
    book_quotes_vectorvectorstore = FAISS.load_local("book_quotes_vectorstore", embeddings, allow_dangerous_deserialization=True)

    chunks_query_retriever = chunks_vector_store.as_retriever(search_kwargs={"k": 1})
    chapter_summaries_query_retriever = chapter_summaries_vector_store.as_retriever(search_kwargs={"k": 1})
    book_quotes_query_retriever = book_quotes_vectorstore.as_retriever(search_kwargs={"k": 10})
    
    return chunks_query_retriever, chapter_summaries_query_retriever, book_quotes_query_retriever
```

| 维度 | 分析 |
|------|------|
| **功能** | 加载 3 个 FAISS 索引并创建检索器 |
| **参数** | 无 |
| **返回值** | 三元组 (chunks_retriever, summaries_retriever, quotes_retriever) |
| **核心逻辑** | 1. 创建 OpenAIEmbeddings → 2. 加载 3 个索引 → 3. 转换为检索器 |
| **关键参数** | `allow_dangerous_deserialization=True` — 允许反序列化（安全风险） |
| **k 值差异** | chunks/summaries 用 k=1（精确匹配），quotes 用 k=10（召回优先） |
| **潜在问题** | ① 索引文件不存在会 FileNotFoundError；② API Key 无效会认证失败 |
| **改进建议** | 添加文件存在检查；支持配置 k 值 |

**k 值设计 Rationale**:
- **chunks k=1**: 每个 chunk 较长（1000 字符），1 个最相关即可
- **summaries k=1**: 章节摘要唯一，1 个最相关章节足够
- **quotes k=10**: 引文较短，需要更多候选

---

### 5.3.2 蒸馏 Chain 部分

#### `create_keep_only_relevant_content_chain()`

```python
def create_keep_only_relevant_content_chain():
    keep_only_relevant_content_prompt_template = """you receive a query: {query} and retrieved docuemnts: {retrieved_documents} from a
    vector store.
    You need to filter out all the non relevant information that don't supply important information regarding the {query}.
    your goal is just to filter out the non relevant information.
    you can remove parts of sentences that are not relevant to the query or remove whole sentences that are not relevant to the query.
    DO NOT ADD ANY NEW INFORMATION THAT IS NOT IN THE RETRIEVED DOCUMENTS.
    output the filtered relevant content.
    """

    class KeepRelevantContent(BaseModel):
        relevant_content: str = Field(description="The relevant content from the retrieved documents that is relevant to the query.")

    keep_only_relevant_content_prompt = PromptTemplate(
        template=keep_only_relevant_content_prompt_template,
        input_variables=["query", "retrieved_documents"],
    )

    keep_only_relevant_content_llm = ChatOpenAI(temperature=0, model_name="gpt-4o", max_tokens=2000)
    keep_only_relevant_content_chain = keep_only_relevant_content_prompt | keep_only_relevant_content_llm.with_structured_output(KeepRelevantContent)
    return keep_only_relevant_content_chain
```

| 维度 | 分析 |
|------|------|
| **功能** | 创建蒸馏 Chain，过滤检索结果中的无关信息 |
| **Prompt 设计** | 强调"只过滤不添加"（DO NOT ADD ANY NEW INFORMATION） |
| **LLM 配置** | temperature=0（确定性），gpt-4o，max_tokens=2000 |
| **输出 Schema** | `KeepRelevantContent.relevant_content: str` |
| **关键约束** | 可以删除句子片段或整个句子，但不能添加新信息 |
| **潜在问题** | ① "DO NOT ADD" 约束可能无法完全保证；② 长文本可能超出 max_tokens |

**Prompt 工程分析**:
```
正向指令: "filter out all the non relevant information"
操作指令: "remove parts of sentences" / "remove whole sentences"
负向约束: "DO NOT ADD ANY NEW INFORMATION"
```

> **设计 Rationale**: 蒸馏的核心目标是**减少噪声**。检索结果通常包含大量无关句子，蒸馏后：
> 1. 减少 LLM 后续处理的 token 数
> 2. 降低幻觉风险（无关信息可能被误用）
> 3. 提高聚合上下文的质量

---

#### `keep_only_relevant_content(state) -> dict`

```python
def keep_only_relevant_content(state):
    question = state["question"]
    context = state["context"]

    input_data = {"query": question, "retrieved_documents": context}
    output = keep_only_relevant_content_chain.invoke(input_data)
    relevant_content = output.relevant_content
    relevant_content = "".join(relevant_content)  # 可能是 list 转 str
    relevant_content = escape_quotes(relevant_content)

    return {"relevant_context": relevant_content, "context": context, "question": question}
```

| 维度 | 分析 |
|------|------|
| **功能** | 执行蒸馏并更新状态 |
| **输入** | state["question"], state["context"] |
| **输出** | 新增 state["relevant_context"]，保留原始 context |
| **特殊处理** | `"".join(relevant_content)` — 处理可能的 list 输出 |
| **潜在问题** | ① 如果 relevant_content 是 str，`"".join()` 会按字符拼接；② 无空值检查 |

---

### 5.3.3 CoT 回答 Chain 部分

#### `create_question_answer_from_context_cot_chain()`

```python
def create_question_answer_from_context_cot_chain():
    class QuestionAnswerFromContext(BaseModel):
        answer_based_on_content: str = Field(description="generates an answer to a query based on a given context.")

    question_answer_from_context_llm = ChatOpenAI(temperature=0, model_name="gpt-4o", max_tokens=2000)

    question_answer_cot_prompt_template = """ 
    Examples of Chain-of-Thought Reasoning

    Example 1
    Context: Mary is taller than Jane. Jane is shorter than Tom. Tom is the same height as David.
    Question: Who is the tallest person?
    Reasoning Chain:
    The context tells us Mary is taller than Jane
    It also says Jane is shorter than Tom
    And Tom is the same height as David
    So the order from tallest to shortest is: Mary, Tom/David, Jane
    Therefore, Mary must be the tallest person

    Example 2
    Context: Harry was reading a book about magic spells...
    Question: Based on the context, if Harry cast these spells, what could he do?
    Reasoning Chain: ...

    Example 3 
    Context: Harry Potter woke up on his birthday to find a present...
    Question: Why did Harry receive a broomstick for his birthday?
    Reasoning Chain:
    The context states that Harry Potter woke up on his birthday and received a present - a Nimbus 2000 broomstick.
    However, the context does not provide any information about why he received that specific present or who gave it to him.
    There are no details about Harry's interests, hobbies, or the person who gifted him the broomstick.
    Without any additional context about Harry's background or the gift-giver's motivations, there is no way to determine the reason he received a broomstick as a birthday present.

    For the question below, provide your answer by first showing your step-by-step reasoning process...
    Context
    {context}
    Question
    {question}
    """
    # ...
```

| 维度 | 分析 |
|------|------|
| **功能** | 创建 Chain-of-Thought 回答 Chain |
| **Prompt 设计** | 3 个示例：2 个正向 + 1 个负例（无法回答） |
| **示例选择** | Example 1: 传递推理；Example 2: 多步推理；Example 3: 拒绝回答 |
| **负例设计** | Example 3 明确教模型"不知道就说不知道" |
| **潜在问题** | ① Prompt 很长（~500 tokens）；② 示例与哈利波特无关可能影响效果 |

**CoT Prompt 设计 Rationale**:

```
Example 1 (简单传递): A>B, B<C, C=D → A 最大
Example 2 (多步聚合): 多个咒语效果 → 综合描述
Example 3 (拒绝回答): 信息不足 → "there is no way to determine"
```

> **关键洞察**: Example 3 是**刻意设计的负例**。它教会模型：
> - 当上下文不足以回答时，应明确承认
> - 不要编造信息来"填补空白"
> - 这是防止幻觉的核心机制之一

---

#### `answer_question_from_context(state) -> dict`

```python
def answer_question_from_context(state):
    question = state["question"]
    context = state["aggregated_context"] if "aggregated_context" in state else state["context"]

    input_data = {"question": question, "context": context}
    output = question_answer_from_context_cot_chain.invoke(input_data)
    answer = output.answer_based_on_content
    return {"answer": answer, "context": context, "question": question}
```

| 维度 | 分析 |
|------|------|
| **功能** | 基于上下文生成答案 |
| **上下文选择** | 优先使用 aggregated_context（多步聚合），否则用 context |
| **核心逻辑** | 调用 CoT Chain 生成推理链 + 答案 |
| **潜在问题** | ① 长上下文可能超出 token 限制；② 无重试机制 |

---

### 5.3.4 验证 Chain 部分

#### `is_answer_grounded_on_context(state) -> str`

```python
def is_answer_grounded_on_context(state):
    context = state["context"]
    answer = state["answer"]
    result = is_grounded_on_facts_chain.invoke({"context": context, "answer": answer})
    grounded_on_facts = result.grounded_on_facts
    if not grounded_on_facts:
        return "hallucination"  # 触发循环
    else:
        return "grounded on context"  # 结束子图
```

| 维度 | 分析 |
|------|------|
| **功能** | 判断答案是否基于上下文（幻觉检测） |
| **返回值** | 字符串路由键："hallucination" 或 "grounded on context" |
| **核心逻辑** | 调用 is_grounded_on_facts_chain，返回 bool |
| **路由作用** | "hallucination" → 循环回 answer_question_from_context 重新生成 |
| **潜在问题** | ① LLM 判断可能有误；② 无限循环风险 |

**幻觉检测 Prompt**:
```python
is_grounded_on_facts_prompt_template = """You are a fact-checker that determines if the given answer {answer} 
is grounded in the given context {context}
you don't mind if it doesn't make sense, as long as it is grounded in the context.
output a json containing the answer to the question, and appart from the json format don't output any additional text."""
```

> **设计 Rationale**: "you don't mind if it doesn't make sense" — 这条指令的意思是：
> - 即使答案在逻辑上不完整，只要基于上下文即可
> - 重点是"基于事实"而非"逻辑完美"

---

### 5.3.5 三路检索函数

#### `retrieve_chunks_context_per_question(state) -> dict`

```python
def retrieve_chunks_context_per_question(state):
    question = state["question"]
    docs = chunks_query_retriever.get_relevant_documents(question)
    context = " ".join(doc.page_content for doc in docs)
    context = escape_quotes(context)
    return {"context": context, "question": question}
```

| 维度 | 分析 |
|------|------|
| **功能** | 从 chunks 向量存储检索相关段落 |
| **检索器** | `chunks_query_retriever` (k=1) |
| **输出** | 拼接后的文档文本（转义引号） |
| **潜在问题** | ① 仅返回 1 个文档，可能遗漏信息；② 无空结果处理 |

#### `retrieve_summaries_context_per_question(state) -> dict`

```python
def retrieve_summaries_context_per_question(state):
    question = state["question"]
    docs_summaries = chapter_summaries_query_retriever.get_relevant_documents(state["question"])
    context_summaries = " ".join(
        f"{doc.page_content} (Chapter {doc.metadata['chapter']})" for doc in docs_summaries
    )
    context_summaries = escape_quotes(context_summaries)
    return {"context": context_summaries, "question": question}
```

| 维度 | 分析 |
|------|------|
| **功能** | 从章节摘要向量存储检索 |
| **特殊处理** | 附加章节号元数据 `(Chapter N)` |
| **输出** | 带引用的摘要文本 |
| **设计 Rationale** | 章节号提供可追溯性，便于验证 |

#### `retrieve_book_quotes_context_per_question(state) -> dict`

```python
def retrieve_book_quotes_context_per_question(state):
    question = state["question"]
    docs_book_quotes = book_quotes_query_retriever.get_relevant_documents(state["question"])
    book_qoutes = " ".join(doc.page_content for doc in docs_book_quotes)
    book_qoutes_context = escape_quotes(book_qoutes)
    return {"context": book_qoutes_context, "question": question}
```

| 维度 | 分析 |
|------|------|
| **功能** | 从引文向量存储检索 |
| **检索器** | `book_quotes_query_retriever` (k=10) |
| **输出** | 拼接的引文文本 |
| **潜在问题** | 10 条引文可能过多，增加后续处理负担 |

---

### 5.3.6 子图创建函数

#### `create_qualitative_retrieval_book_chunks_workflow_app()`

```python
def create_qualitative_retrieval_book_chunks_workflow_app():
    qualitative_chunks_retrieval_workflow = StateGraph(QualitativeRetrievalGraphState)

    qualitative_chunks_retrieval_workflow.add_node("retrieve_chunks_context_per_question", retrieve_chunks_context_per_question)
    qualitative_chunks_retrieval_workflow.add_node("keep_only_relevant_content", keep_only_relevant_content)

    qualitative_chunks_retrieval_workflow.set_entry_point("retrieve_chunks_context_per_question")
    qualitative_chunks_retrieval_workflow.add_edge("retrieve_chunks_context_per_question", "keep_only_relevant_content")

    qualitative_chunks_retrieval_workflow.add_conditional_edges(
        "keep_only_relevant_content",
        is_distilled_content_grounded_on_content,
        {"grounded on the original context": END,
         "not grounded on the original context": "keep_only_relevant_content"},
    )

    qualitative_chunks_retrieval_workflow_app = qualitative_chunks_retrieval_workflow.compile()
    return qualitative_chunks_retrieval_workflow_app
```

| 维度 | 分析 |
|------|------|
| **功能** | 创建 chunks 检索子图 |
| **状态类型** | `QualitativeRetrievalGraphState` (question, context, relevant_context) |
| **节点** | 2 个：retrieve + keep_relevant |
| **边** | 1 条无条件 + 1 条条件（循环或结束） |
| **循环机制** | 蒸馏未通过验证 → 重新蒸馏 |

**子图结构**:
```
retrieve_chunks_context_per_question
        │
        ▼
keep_only_relevant_content
        │
        ├── grounded → END
        └── not grounded → keep_only_relevant_content (循环)
```

> **设计 Rationale**: 子图共享 `keep_only_relevant_content` 和 `is_distilled_content_grounded_on_content`：
> - 代码复用
> - 一致的蒸馏标准
> - 但不同数据源可能需要不同蒸馏策略

---

### 5.3.7 计划-执行核心函数

#### `create_plan_chain()`

```python
def create_plan_chain():
    planner_prompt =""" For the given query {question}, come up with a simple step by step plan of how to figure out the answer. 
    This plan should involve individual tasks, that if executed correctly will yield the correct answer. 
    Do not add any superfluous steps. 
    The result of the final step should be the final answer. 
    Make sure that each step has all the information needed - do not skip steps.
    """

    planner_prompt = PromptTemplate(template=planner_prompt, input_variables=["question"])
    planner_llm = ChatOpenAI(temperature=0, model_name="gpt-4o", max_tokens=2000)
    planner = planner_prompt | planner_llm.with_structured_output(Plan)
    return planner
```

| 维度 | 分析 |
|------|------|
| **功能** | 创建计划生成 Chain |
| **Prompt 约束** | ① 简单步骤；② 不多余；③ 最终步骤=最终答案；④ 不跳过步骤 |
| **输出 Schema** | `Plan(steps: List[str])` |
| **潜在问题** | ① 无示例（zero-shot）；② 复杂问题可能生成不完整计划 |

---

#### `create_task_handler_chain()`

```python
def create_task_handler_chain():
    tasks_handler_prompt_template = """You are a task handler that receives a task {curr_task} and have to decide with tool to use to execute the task.
    You have the following tools at your disposal:
    Tool A: a tool that retrieves relevant information from a vector store of book chunks based on a given query.
    - use Tool A when you think the current task should search for information in the book chunks.
    Took B: a tool that retrieves relevant information from a vector store of chapter summaries based on a given query.
    - use Tool B when you think the current task should search for information in the chapter summaries.
    Tool C: a tool that retrieves relevant information from a vector store of quotes from the book based on a given query.
    - use Tool C when you think the current task should search for information in the book quotes.
    Tool D: a tool that answers a question from a given context.
    - use Tool D ONLY when you the current task can be answered by the aggregated context {aggregated_context}

    you also receive the last tool used {last_tool}
    if {last_tool} was retrieve_chunks, use other tools than Tool A.

    You also have the past steps {past_steps} that you can use to make decisions and understand the context of the task.
    You also have the initial user's question {question} that you can use to make decisions and understand the context of the task.
    if you decide to use Tools A,B or C, output the query to be used for the tool and also output the relevant tool.
    if you decide to use Tool D, output the question to be used for the tool, the context, and also that the tool to be used is Tool D.
    """

    class TaskHandlerOutput(BaseModel):
        query: str = Field(description="...")
        curr_context: str = Field(description="...")
        tool: str = Field(description="...should be either retrieve_chunks, retrieve_summaries, retrieve_quotes, or answer_from_context.")

    task_handler_prompt = PromptTemplate(
        template=tasks_handler_prompt_template,
        input_variables=["curr_task", "aggregated_context", "last_tool" "past_steps", "question"],  # 注意：缺少逗号！
    )
    task_handler_llm = ChatOpenAI(temperature=0, model_name="gpt-4o", max_tokens=2000)
    task_handler_chain = task_handler_prompt | task_handler_llm.with_structured_output(TaskHandlerOutput)
    return task_handler_chain
```

| 维度 | 分析 |
|------|------|
| **功能** | 创建任务路由 Chain |
| **工具选择逻辑** | 4 种工具 (A/B/C/D)，基于任务类型选择 |
| **防重复机制** | "if last_tool was retrieve_chunks, use other tools than Tool A" |
| **输出 Schema** | `TaskHandlerOutput(query, curr_context, tool)` |
| **⚠️ BUG** | `input_variables` 中 `"last_tool" "past_steps"` 缺少逗号，Python 会隐式拼接为 `"last_toolpast_steps"` |

**BUG 分析**:
```python
input_variables=["curr_task", "aggregated_context", "last_tool" "past_steps", "question"]
#                                                 ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
# Python 隐式字符串拼接 → "last_toolpast_steps"
# 这意味着 past_steps 变量无法被正确注入！
```

> **严重程度**: 中等。`past_steps` 变量无法注入，LLM 缺少历史上下文，可能影响决策质量。

---

#### `create_anonymize_question_chain()`

```python
def create_anonymize_question_chain():
    class AnonymizeQuestion(BaseModel):
        anonymized_question: str = Field(description="Anonymized question.")
        mapping: dict = Field(description="Mapping of original name entities to variables.")
        explanation: str = Field(description="Explanation of the action.")

    anonymize_question_parser = JsonOutputParser(pydantic_object=AnonymizeQuestion)

    anonymize_question_prompt_template = """ You are a question anonymizer. The input You receive is a string containing several words that
    construct a question {question}. Your goal is to changes all name entities in the input to variables, and remember the mapping of the original name entities to the variables.
    ```example1:
            if the input is \"who is harry potter?\" the output should be \"who is X?\" and the mapping should be {{\"X\": \"harry potter\"}} ```
    ```example2:
            if the input is \"how did the bad guy played with the alex and rony?\"
            the output should be \"how did the X played with the Y and Z?\" and the mapping should be {{\"X\": \"bad guy\", \"Y\": \"alex\", \"Z\": \"rony\"}}```
    you must replace all name entities in the input with variables, and remember the mapping of the original name entities to the variables.
    output the anonymized question and the mapping as two separate fields in a json format as described here, without any additional text apart from the json format.
   """

    anonymize_question_prompt = PromptTemplate(
        template=anonymize_question_prompt_template,
        input_variables=["question"],
        partial_variables={"format_instructions": anonymize_question_parser.get_format_instructions()},
    )

    anonymize_question_llm = ChatOpenAI(temperature=0, model_name="gpt-4o", max_tokens=2000)
    anonymize_question_chain = anonymize_question_prompt | anonymize_question_llm | anonymize_question_parser
    return anonymize_question_chain
```

| 维度 | 分析 |
|------|------|
| **功能** | 创建问题匿名化 Chain |
| **Chain 结构** | Prompt → LLM → JsonOutputParser（注意：不是 with_structured_output） |
| **输出 Schema** | `AnonymizeQuestion(anonymized_question, mapping, explanation)` |
| **示例驱动** | 2 个示例明确展示替换规则 |
| **格式注入** | 使用 `partial_variables` 注入 JSON 格式指令 |
| **潜在问题** | ① 转义复杂（`\"` 和 `{{}}`）；② 实体识别依赖 LLM 理解 |

**与其他 Chain 的差异**:
- 其他 Chain 使用 `llm.with_structured_output(Schema)`
- 此 Chain 使用 `llm | JsonOutputParser(pydantic_object=Schema)`
- 两种方式效果类似，但实现不同

---

### 5.3.8 主图构建函数

#### `create_agent()`

```python
def create_agent():
    agent_workflow = StateGraph(PlanExecute)

    # 添加节点
    agent_workflow.add_node("anonymize_question", anonymize_queries)
    agent_workflow.add_node("planner", plan_step)
    agent_workflow.add_node("break_down_plan", break_down_plan_step)
    agent_workflow.add_node("de_anonymize_plan", deanonymize_queries)
    agent_workflow.add_node("retrieve_chunks", run_qualitative_chunks_retrieval_workflow)
    agent_workflow.add_node("retrieve_summaries", run_qualitative_summaries_retrieval_workflow)
    agent_workflow.add_node("retrieve_book_quotes", run_qualitative_book_quotes_retrieval_workflow)
    agent_workflow.add_node("answer", run_qualtative_answer_workflow)
    agent_workflow.add_node("task_handler", run_task_handler_chain)
    agent_workflow.add_node("replan", replan_step)
    agent_workflow.add_node("get_final_answer", run_qualtative_answer_workflow_for_final_answer)

    # 设置入口
    agent_workflow.set_entry_point("anonymize_question")

    # 计划流水线边
    agent_workflow.add_edge("anonymize_question", "planner")
    agent_workflow.add_edge("planner", "de_anonymize_plan")
    agent_workflow.add_edge("de_anonymize_plan", "break_down_plan")
    agent_workflow.add_edge("break_down_plan", "task_handler")

    # 任务路由条件边
    agent_workflow.add_conditional_edges("task_handler", retrieve_or_answer, {
        "chosen_tool_is_retrieve_chunks": "retrieve_chunks",
        "chosen_tool_is_retrieve_summaries": "retrieve_summaries",
        "chosen_tool_is_retrieve_quotes": "retrieve_book_quotes",
        "chosen_tool_is_answer": "answer"
    })

    # 检索/回答后回到 replan
    agent_workflow.add_edge("retrieve_chunks", "replan")
    agent_workflow.add_edge("retrieve_summaries", "replan")
    agent_workflow.add_edge("retrieve_book_quotes", "replan")
    agent_workflow.add_edge("answer", "replan")

    # Replan 条件边
    agent_workflow.add_conditional_edges("replan", can_be_answered, {
        "can_be_answered_already": "get_final_answer",
        "cannot_be_answered_yet": "break_down_plan"
    })

    # 最终答案
    agent_workflow.add_edge("get_final_answer", END)

    plan_and_execute_app = agent_workflow.compile()
    return plan_and_execute_app
```

| 维度 | 分析 |
|------|------|
| **功能** | 构建完整的 LangGraph 主图 |
| **节点数** | 11 个 |
| **边数** | 10 条无条件 + 2 条条件 |
| **入口** | `anonymize_question` |
| **出口** | `get_final_answer` → END |
| **循环** | `break_down_plan → task_handler → ... → replan → break_down_plan` |

**图结构可视化**:
```
anonymize_question → planner → de_anonymize_plan → break_down_plan → task_handler
                                                                          │
                                                    ┌─────────────────────┤
                                                    │         │           │
                                                    ▼         ▼           ▼
                                              retrieve_chunks  retrieve_summaries  retrieve_book_quotes  answer
                                                    │         │           │            │
                                                    └─────────────────────┤
                                                                          │
                                                                          ▼
                                                                       replan
                                                                          │
                                                    ┌─────────────────────┤
                                                    │                     │
                                                    ▼                     ▼
                                          get_final_answer          break_down_plan
                                                    │
                                                    ▼
                                                   END
```

---

### 5.3.9 子图执行函数

#### `run_qualitative_chunks_retrieval_workflow(state) -> dict`

```python
def run_qualitative_chunks_retrieval_workflow(state):
    state["curr_state"] = "retrieve_chunks"
    question = state["query_to_retrieve_or_answer"]
    inputs = {"question": question}
    
    for output in qualitative_chunks_retrieval_workflow_app.stream(inputs):
        for _, _ in output.items():
            pass  # 仅消费流，不处理中间输出
    
    if not state["aggregated_context"]:
        state["aggregated_context"] = ""
    state["aggregated_context"] += output['relevant_context']
    return state
```

| 维度 | 分析 |
|------|------|
| **功能** | 执行 chunks 检索子图并聚合结果 |
| **流式处理** | 使用 `stream()` 消费子图输出 |
| **结果聚合** | 将 `relevant_context` 追加到 `aggregated_context` |
| **⚠️ 潜在问题** | `output` 变量在循环结束后是**最后一个**输出，如果子图有多个输出，可能不是期望的 |

**BUG 分析**:
```python
for output in qualitative_chunks_retrieval_workflow_app.stream(inputs):
    for _, _ in output.items():
        pass
# 循环结束后，output 是最后一个 yield 的值
state["aggregated_context"] += output['relevant_context']
```

> **问题**: LangGraph 的 `stream()` 会 yield 每个节点的输出。如果子图有 2 个节点（retrieve + keep_relevant），`output` 最后一次是 `keep_relevant` 的输出。但如果循环逻辑改变，可能取到错误的输出。

---

## 5.4 `simulate_agent.py` 逐函数走读

### 5.4.1 `create_network_graph(current_state: str) -> Network`

```python
def create_network_graph(current_state):
    net = Network(directed=True, notebook=True, height="250px", width="100%")
    net.toggle_physics(False)
    
    nodes = [
        {"id": "anonymize_question", "label": "anonymize_question", "x": 0, "y": 0},
        {"id": "planner", "label": "planner", "x": 175*1.75, "y": -100},
        # ... 更多节点（硬编码坐标）
    ]
    
    edges = [
        ("anonymize_question", "planner"),
        # ... 更多边
    ]
    
    for node in nodes:
        color = "#00FF00" if node["id"] == current_state else "#FF69B4"
        net.add_node(node["id"], label=node["label"], x=node["x"], y=node["y"], color=color, physics=False, font={'size': 22})
    
    for edge in edges:
        net.add_edge(edge[0], edge[1], color="#808080")
    
    net.options.edges.smooth.type = "straight"
    net.options.edges.width = 1.5
    
    return net
```

| 维度 | 分析 |
|------|------|
| **功能** | 创建 pyvis 网络图，当前节点绿色高亮 |
| **节点坐标** | 硬编码（`x`, `y`），与主图结构**不同步** |
| **颜色逻辑** | 当前节点 `#00FF00`（绿），其他 `#FF69B4`（粉） |
| **潜在问题** | ① 节点/边与主图不同步；② 坐标计算 `175*1.75` 不直观 |

---

### 5.4.2 `execute_plan_and_print_steps(inputs, plan_and_execute_app, placeholders, graph_placeholder, recursion_limit=25) -> str`

```python
def execute_plan_and_print_steps(inputs, plan_and_execute_app, placeholders, graph_placeholder, recursion_limit=25):
    config = {"recursion_limit": recursion_limit}
    agent_state_value = None
    progress_bar = st.progress(0)
    step = 0
    previous_state = None
    previous_values = {key: None for key in placeholders}

    try:
        for plan_output in plan_and_execute_app.stream(inputs, config=config):
            step += 1
            for _, agent_state_value in plan_output.items():
                previous_values, previous_state = update_placeholders_and_graph(
                    agent_state_value, placeholders, graph_placeholder, previous_values, previous_state
                )
                progress_bar.progress(step / recursion_limit)
                if step >= recursion_limit:
                    break

        # 最终更新
        for key, placeholder in placeholders.items():
            if key in previous_values and previous_values[key] is not None:
                if isinstance(previous_values[key], list):
                    formatted_value = "\n".join([f"{i+1}. {item}" for i, item in enumerate(previous_values[key])])
                else:
                    formatted_value = previous_values[key]
                placeholder.markdown(f"{formatted_value}")

        response = agent_state_value.get('response', "No response found.") if agent_state_value else "No response found."
    except Exception as e:
        response = f"An error occurred: {str(e)}"
        st.error(f"Error: {e}")

    return response
```

| 维度 | 分析 |
|------|------|
| **功能** | 执行 Agent 并实时更新 UI |
| **流式处理** | 使用 `stream()` 消费主图输出 |
| **进度条** | `step / recursion_limit` 估算进度 |
| **异常处理** | try-except 捕获所有异常，返回错误信息 |
| **潜在问题** | ① 进度条是线性的，实际步数不可预测；② 异常处理过于宽泛 |

---

### 5.4.3 `main()`

```python
def main():
    st.set_page_config(layout="wide")
    st.title("Real-Time Agent Execution Visualization")
    
    plan_and_execute_app = create_agent()
    question = st.text_input("Enter your question:", "what is the class that the proffessor who helped the villain is teaching?")

    if st.button("Run Agent"):
        inputs = {"question": question}
        
        st.markdown("**Graph**")
        graph_placeholder = st.empty()

        col1, col2, col3 = st.columns([1, 1, 4])
        
        with col1:
            st.markdown("**Plan**")
        with col2:
            st.markdown("**Past Steps**")
        with col3:
            st.markdown("**Aggregated Context**")

        placeholders = {
            "plan": col1.empty(),
            "past_steps": col2.empty(),
            "aggregated_context": col3.empty(),
        }

        response = execute_plan_and_print_steps(inputs, plan_and_execute_app, placeholders, graph_placeholder, recursion_limit=45)
        st.write("Final Answer:")
        st.write(response)
```

| 维度 | 分析 |
|------|------|
| **功能** | Streamlit 应用入口 |
| **布局** | 宽布局，1 行图 + 3 列信息 |
| **默认问题** | "what is the class that the proffessor who helped the villain is teaching?" |
| **递归限制** | 45（比默认 25 更高） |
| **潜在问题** | ① 每次点击都重新创建 Agent；② 无输入校验 |

---

## 5.5 代码质量评估

### 5.5.1 优点

| 维度 | 评价 | 示例 |
|------|------|------|
| **模块化** | ⭐⭐⭐⭐ | 三个文件职责清晰 |
| **类型注解** | ⭐⭐⭐⭐ | TypedDict + Pydantic 广泛使用 |
| **函数粒度** | ⭐⭐⭐⭐ | 每个函数职责单一 |
| **Prompt 设计** | ⭐⭐⭐⭐⭐ | CoT 示例精心设计 |
| **图编排** | ⭐⭐⭐⭐⭐ | LangGraph 使用规范 |

### 5.5.2 缺陷与 BUG

| 严重度 | 问题 | 位置 | 影响 |
|--------|------|------|------|
| 🔴 高 | `allow_dangerous_deserialization=True` | :36-38 | 反序列化安全风险 |
| 🟡 中 | `input_variables` 缺少逗号 | :706 | past_steps 无法注入 |
| 🟡 中 | `import *` 通配符导入 | simulate_agent.py:5 | 命名空间污染 |
| 🟡 中 | 模块级单例 | :47, :800 | 难以测试 |
| 🟢 低 | 硬编码节点坐标 | simulate_agent.py:22-33 | 与图结构不同步 |
| 🟢 低 | 无 docstring | 部分函数 | 可读性下降 |
| 🟢 低 | print 调试输出 | 多处 | 生产环境不适用 |

### 5.5.3 性能瓶颈

| 瓶颈 | 位置 | 影响 | 优化建议 |
|------|------|------|----------|
| **LLM 调用次数** | 每问题 10-30 次 | 高延迟 | 缓存、批处理 |
| **串行执行** | 主图循环 | 无法并行 | 并行检索 |
| **全量图重渲染** | 每次状态变化 | UI 延迟 | 增量更新 |
| **FAISS 加载** | 启动时 | 启动延迟 | 懒加载 |

---

## 5.6 本章小结

本章对三个核心文件进行了**逐函数深度走读**：

1. **helper_functions.py**: 11 个工具函数，覆盖文本处理、PDF 加载、相似度计算
2. **functions_for_pipeline.py**: ~40 个函数，包括 12 条 Chain、4 个子图、1 个主图
3. **simulate_agent.py**: 6 个函数，实现 Streamlit UI 和图可视化

**关键发现**:
- ✅ 整体架构清晰，模块化良好
- ✅ Prompt 设计精心（CoT 示例、负例）
- ⚠️ 存在 1 个 BUG（input_variables 缺少逗号）
- ⚠️ 安全风险（反序列化）
- 🔧 可改进（模块级单例、import *、硬编码坐标）

**下一章**: [06-data-model.md](./06-data-model.md) — 深入分析数据模型与向量存储设计。

---

☕️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../res/promotion.jpg)