# 架构决策记录 (ADR)

> 本文档记录了 FsExplorer 项目中的关键架构决策。每个决策都包含背景、方案选择、权衡分析和后果评估。

---

## ADR-001: 选择 Google Gemini 作为 LLM 提供商

**Status**: Accepted
**Date**: 2026-01

### Context

FsExplorer 需要一个 LLM 来驱动 Agent 的探索决策。选型时需满足以下核心需求：

1. **结构化 JSON 输出**：Agent 需要输出结构化的 Action 对象（包含工具调用、参数等），LLM 必须支持可靠的 JSON 模式输出。
2. **工具调用（Tool Use）**：LLM 需要能够根据上下文选择合适的工具并生成调用参数。
3. **长上下文窗口**：探索过程中需要累积大量文档内容和工具结果，需要足够大的上下文窗口。
4. **成本效益**：作为需要频繁调用 API 的 Agent 系统，单位 token 成本是重要考量因素。
5. **响应速度**：用户体验要求 LLM 能够快速返回结果，减少等待时间。

### Decision

选择 **Google Gemini 3 Flash**（`gemini-3-flash-preview`）作为默认 LLM 提供商。

核心配置：

```python
# agent.py
MODEL_NAME = "gemini-3-flash-preview"
GENERATION_CONFIG = {
    "response_schema": Action,  # Pydantic 模型定义的结构化输出
    "temperature": 0.1,         # 低温度确保输出稳定性
}
```

### Alternatives

| 方案 | 优势 | 劣势 | 未选择原因 |
|------|------|------|-----------|
| **OpenAI GPT-4o** | 生态成熟，工具调用稳定 | 成本较高，JSON 模式需额外配置 | 成本是 Gemini Flash 的 3-5 倍 |
| **Anthropic Claude 3.5 Sonnet** | 长上下文表现优秀，推理能力强 | API 速率限制较严格，成本中等 | 结构化输出不如 Gemini 原生支持方便 |
| **本地 LLM (Llama 3)** | 无 API 成本，数据不出本地 | 需要 GPU 资源，工具调用可靠性差 | 结构化输出质量不稳定，部署复杂 |
| **Google Gemini Pro** | 能力更强 | 成本更高，速度较慢 | Flash 已满足需求，无需 Pro 级别能力 |

### Consequences

**正面影响：**

- **快速低成本**：Gemini Flash 是市场上性价比最高的 LLM 之一，适合高频 Agent 调用场景。
- **原生 JSON 模式**：通过 `response_schema` 参数直接指定 Pydantic 模型，无需额外的提示词工程即可确保输出格式正确。
- **长上下文**：支持 1M+ token 的上下文窗口，足以容纳大型文档集的内容。
- **工具调用支持**：原生支持 function calling，Agent 可以自然地选择和调用工具。

**负面影响：**

- **供应商锁定**：代码中硬编码了 Gemini 特定的 API 调用方式，切换到其他提供商需要修改多处代码。
- **API Key 依赖**：需要用户配置 Google API Key，增加了上手门槛。
- **地区可用性**：某些地区可能无法稳定访问 Google API。
- **模型更新风险**：Google 可能随时更新或弃用特定模型版本。

### References

- `src/fs_explorer/agent.py` — LLM 客户端配置和调用逻辑
- `src/fs_explorer/models.py` — Action Pydantic 模型定义
- Google AI Studio: https://aistudio.google.com/

---

## ADR-002: 事件驱动 Workflow 架构

**Status**: Accepted
**Date**: 2026-01

### Context

FsExplorer 的 Agent 需要执行多步骤的探索流程：接收任务 → 选择工具 → 执行工具 → 分析结果 → 决定下一步 → 生成答案。这个流程需要：

1. **流式输出**：用户应该能够实时看到 Agent 的执行进度，而不是等待全部完成后一次性显示。
2. **步骤可观测**：每个步骤的状态变化应该可追踪、可记录，便于调试和分析。
3. **Human-in-the-Loop**：某些场景下需要向用户提问，等待用户回答后继续执行。
4. **可扩展性**：未来可能需要添加新的步骤类型（如"验证答案"、"生成报告"）。

### Decision

采用 **llama-index-workflows** 的事件驱动架构，将探索流程分解为离散的步骤和事件。

核心设计：

```python
# workflow.py
class FsExplorerWorkflow(Workflow):
    
    @step
    async def start_exploration(self, ev: InputEvent) -> ToolCallEvent | StopEvent:
        # 初始化探索，发送第一个 LLM 请求
        ...
    
    @step
    async def tool_call_action(self, ev: ToolCallEvent) -> ToolResultEvent:
        # 执行工具调用
        ...
    
    @step
    async def go_deeper_action(self, ev: GoDeeperEvent) -> ToolCallEvent:
        # 进入子目录继续探索
        ...
    
    @step
    async def receive_human_answer(self, ev: HumanAnswerEvent) -> ToolCallEvent:
        # 接收用户回答，继续探索
        ...
```

事件类型定义：

- `InputEvent` — 用户输入的查询任务
- `ToolCallEvent` — Agent 决定调用工具
- `ToolResultEvent` — 工具执行完成，返回结果
- `GoDeeperEvent` — Agent 决定深入某个目录
- `AskHumanEvent` — Agent 需要向用户提问
- `HumanAnswerEvent` — 用户回答
- `ExplorationEndEvent` — 探索完成，包含最终结果

### Alternatives

| 方案 | 优势 | 劣势 | 未选择原因 |
|------|------|------|-----------|
| **纯 async 循环** | 实现简单，无外部依赖 | 难以实现流式输出和步骤追踪 | 无法满足可观测性需求 |
| **LangGraph** | 生态成熟，社区活跃 | 抽象层次过多，学习曲线陡峭 | 对于当前需求过于重量级 |
| **自研状态机** | 完全可控，无外部依赖 | 需要自行实现事件分发和流式支持 | 重复造轮子，维护成本高 |
| **Celery 任务队列** | 支持分布式执行 | 引入消息队列依赖，架构复杂 | 当前阶段不需要分布式 |

### Consequences

**正面影响：**

- **解耦**：Workflow 层、Agent 层、工具层通过事件总线解耦，每层可以独立演进。
- **可扩展步骤**：新增步骤只需添加 `@step` 装饰器方法，无需修改现有流程。
- **流式输出**：事件天然支持流式处理，WebSocket 推送和终端渲染都很自然。
- **Human-in-the-Loop**：`AskHumanEvent` / `HumanAnswerEvent` 机制天然支持人机协作。
- **可测试性**：每个步骤可以独立测试，事件序列可以录制和回放。

**负面影响：**

- **学习曲线**：开发者需要理解 llama-index-workflows 的事件驱动模型。
- **调试难度**：异步事件流的调试比同步代码更困难。
- **依赖风险**：llama-index-workflows 是相对较新的库，API 可能不稳定。

### References

- `src/fs_explorer/workflow.py` — Workflow 实现
- llama-index-workflows 文档: https://docs.llamaindex.dev/en/stable/module_guides/workflow/

---

## ADR-003: DuckDB 作为嵌入式存储

**Status**: Accepted
**Date**: 2026-01

### Context

FsExplorer 的索引模式需要持久化存储以下数据：

1. **文档元数据**：文件路径、修改时间、文档类型、提取的元数据等。
2. **文本块（Chunks）**：文档被分割后的文本片段，用于语义检索。
3. **嵌入向量（Embeddings）**：文本块的向量表示，用于相似度搜索。
4. **Schema 信息**：自动发现的元数据字段定义。

选型时的核心考量：

- **零配置**：用户不应需要安装、配置或管理独立的数据库服务。
- **性能**：需要支持高效的 JSON 路径查询和向量相似度搜索。
- **单文件**：索引应该存储在单个文件中，便于备份和迁移。
- **Python 原生**：需要与 Python 生态无缝集成。

### Decision

选择 **DuckDB** 作为嵌入式存储引擎。

核心表结构：

```sql
-- storage/duckdb.py
CREATE TABLE corpora (
    corpus_id VARCHAR PRIMARY KEY,
    name VARCHAR,
    root_path VARCHAR,
    created_at TIMESTAMP
);

CREATE TABLE documents (
    doc_id VARCHAR PRIMARY KEY,
    corpus_id VARCHAR,
    relative_path VARCHAR,
    file_name VARCHAR,
    extension VARCHAR,
    file_size BIGINT,
    mtime DOUBLE,
    metadata JSON,
    FOREIGN KEY (corpus_id) REFERENCES corpora(corpus_id)
);

CREATE TABLE chunks (
    chunk_id VARCHAR PRIMARY KEY,
    doc_id VARCHAR,
    position INTEGER,
    start_char INTEGER,
    end_char INTEGER,
    text VARCHAR,
    embedding FLOAT[384],
    FOREIGN KEY (doc_id) REFERENCES documents(doc_id)
);

CREATE TABLE schemas (
    schema_id VARCHAR PRIMARY KEY,
    corpus_id VARCHAR,
    fields JSON,
    discovered_at TIMESTAMP
);
```

### Alternatives

| 方案 | 优势 | 劣势 | 未选择原因 |
|------|------|------|-----------|
| **SQLite** | 最广泛使用的嵌入式数据库 | 不支持向量搜索，JSON 查询性能一般 | 需要额外引入向量数据库 |
| **PostgreSQL + pgvector** | 功能最完整，支持向量搜索 | 需要独立的服务进程，配置复杂 | 违背零配置原则 |
| **Chroma** | 专为向量搜索设计 | 不支持结构化元数据查询 | 需要与关系型数据库配合使用 |
| **Qdrant** | 高性能向量搜索 | 需要独立的服务进程，Rust 实现 | 部署复杂，Python 集成不如 DuckDB 原生 |
| **FAISS** | Meta 开发，向量搜索性能极致 | 纯向量库，无结构化查询能力 | 需要与其他存储方案配合 |

### Consequences

**正面影响：**

- **零配置部署**：无需安装、启动或管理数据库服务，`pip install duckdb` 即可使用。
- **单文件存储**：整个索引存储在单个 `.duckdb` 文件中，便于备份、迁移和版本控制。
- **高性能分析**：列式存储引擎对聚合查询和 JSON 路径过滤有天然优势。
- **VSS 扩展**：通过 HNSW 索引支持向量相似度搜索，无需引入额外的向量数据库。
- **Python 原生**：DuckDB 的 Python  API 与 Pandas 无缝集成，数据处理非常方便。

**负面影响：**

- **单写限制**：DuckDB 同一时间只允许一个写连接，并发写入需要额外的同步机制。
- **VSS 扩展可用性**：VSS 扩展在某些平台或 DuckDB 版本上可能不可用，需要降级处理。
- **生态成熟度**：相比 PostgreSQL，DuckDB 的社区和工具生态仍在发展中。
- **备份复杂性**：单文件存储在写入过程中如果崩溃，可能导致文件损坏。

### References

- `src/fs_explorer/storage/duckdb.py` — DuckDB 存储实现
- DuckDB 官方文档: https://duckdb.org/docs/
- DuckDB VSS 扩展: https://duckdb.org/docs/extensions/vss

---

## ADR-004: 三阶段文档探索策略

**Status**: Accepted
**Date**: 2026-01

### Context

FsExplorer 的 Agent 需要高效地探索大量文档来回答用户查询。直接对所有文档进行完整解析（使用 Docling 转换为 Markdown）的成本非常高：

- **时间成本**：解析一个 50 页的 PDF 可能需要 10-30 秒。
- **Token 成本**：解析后的文本会消耗大量 LLM token。
- **信息过载**：将大量无关内容发送给 LLM 会降低决策质量。

因此，需要一个策略来平衡"探索覆盖率"和"资源消耗"。

### Decision

设计**三阶段探索策略**，在 SYSTEM_PROMPT 中明确定义：

**Phase 1：并行扫描（Parallel Scan）**

- 快速浏览目录结构，识别文件类型和基本元数据。
- 使用 `scan_folder` 工具获取文件列表，不进行内容解析。
- 根据文件名、路径、扩展名等元数据初步筛选潜在相关文档。

**Phase 2：深入解析（Deep Dive）**

- 对 Phase 1 筛选出的高价值目标进行完整解析。
- 使用 `parse_document` 工具调用 Docling 将文档转换为 Markdown。
- 分析文档内容，提取关键信息。

**Phase 3：回溯交叉引用（Backtrack Cross-reference）**

- 根据 Phase 2 发现的信息，追踪引用关系。
- 如果文档 A 提到了"详见附录 B"，则去查找附录 B。
- 确保答案的完整性和准确性。

### Alternatives

| 方案 | 优势 | 劣势 | 未选择原因 |
|------|------|------|-----------|
| **全部解析** | 不会遗漏信息 | Token 消耗巨大，速度极慢 | 成本不可接受 |
| **仅关键词搜索** | 速度快，成本低 | 无法理解语义，召回率差 | 无法满足复杂查询需求 |
| **随机探索** | 实现简单 | 效率低下，结果不可靠 | 缺乏方向性，浪费资源 |
| **预过滤 + 解析** | 平衡效率和质量 | 过滤条件可能误判 | 与三阶段策略类似，但缺少回溯机制 |

### Consequences

**正面影响：**

- **节省 Token**：避免对无关文档进行昂贵的解析，实测可节省 60-80% 的 API 调用成本。
- **引用完整性**：Phase 3 的回溯机制确保不会遗漏被引用的关键信息。
- **渐进式披露**：用户可以在任何阶段看到部分结果，不必等待全部完成。
- **灵活调整**：Agent 可以根据初步发现动态调整探索策略。

**负面影响：**

- **提示词复杂**：SYSTEM_PROMPT 需要详细描述三阶段策略，增加了提示词长度和复杂度。
- **依赖 LLM 判断力**：Phase 1 的筛选质量高度依赖 LLM 的判断能力，可能误判。
- **最坏情况**：如果所有文档都相关，三阶段策略可能比直接解析更慢（因为多了筛选步骤）。

### References

- `src/fs_explorer/agent.py` — SYSTEM_PROMPT 中的策略描述
- `src/fs_explorer/fs.py` — scan_folder 和 parse_document 实现

---

## ADR-005: 模块级全局状态管理索引上下文

**Status**: Accepted（待重构）
**Date**: 2026-01

### Context

FsExplorer 的工具函数（如 `semantic_search`、`get_document`）需要访问当前的索引状态，包括：

- **DuckDB 连接**：用于执行查询的数据库连接。
- **Corpus ID**：当前操作的语料库标识。
- **嵌入模型配置**：用于计算查询向量的模型参数。

这些状态在 Agent 初始化时确定，在整个探索过程中保持不变。需要一种方式让工具函数访问这些状态。

### Decision

使用**模块级变量**存储索引上下文，通过 `set_index_context()` / `clear_index_context()` 进行设置和清理。

```python
# agent.py
from dataclasses import dataclass
from typing import Optional

@dataclass
class IndexContext:
    db_path: str
    corpus_id: str
    embedding_model: Optional[str] = None

# 模块级全局变量
_INDEX_CONTEXT: Optional[IndexContext] = None
_EMBEDDING_PROVIDER: Optional[object] = None

def set_index_context(db_path: str, corpus_id: str, embedding_model: Optional[str] = None):
    """设置全局索引上下文。"""
    global _INDEX_CONTEXT, _EMBEDDING_PROVIDER
    _INDEX_CONTEXT = IndexContext(db_path, corpus_id, embedding_model)
    if embedding_model:
        _EMBEDDING_PROVIDER = _create_embedding_provider(embedding_model)

def clear_index_context():
    """清理全局索引上下文。"""
    global _INDEX_CONTEXT, _EMBEDDING_PROVIDER
    _INDEX_CONTEXT = None
    _EMBEDDING_PROVIDER = None

def get_index_context() -> Optional[IndexContext]:
    """获取当前索引上下文。"""
    return _INDEX_CONTEXT
```

工具函数通过访问模块级变量获取上下文：

```python
def semantic_search(query: str, limit: int = 10) -> str:
    ctx = get_index_context()
    if ctx is None:
        return "Error: No index context set. Please run 'index' first."
    # 使用 ctx.db_path 和 ctx.corpus_id 执行查询
    ...
```

### Alternatives

| 方案 | 优势 | 劣势 | 未选择原因 |
|------|------|------|-----------|
| **依赖注入** | 显式依赖，线程安全 | 需要修改所有工具函数的签名 | 工具函数签名会过于复杂 |
| **全局单例类** | 封装性好，可添加方法 | 仍然是全局状态，线程安全问题依旧 | 没有本质改善 |
| **闭包/偏函数** | 无全局状态 | 需要修改工具注册机制 | 与现有 TOOLS 字典模式不兼容 |
| **contextvars** | 线程安全，异步安全 | Python 3.7+ 特性，理解成本略高 | 当前阶段优先简单实现 |

### Consequences

**正面影响：**

- **简单直接**：实现简单，无需复杂的依赖注入框架。
- **向后兼容**：工具函数签名不变，不影响现有代码。
- **易于理解**：开发者可以快速理解状态管理方式。

**负面影响：**

- **非线程安全**：多线程并发请求可能互相覆盖状态，导致查询发送到错误的数据库。
- **测试困难**：测试之间可能因状态残留而相互影响，必须显式调用 `clear_index_context()`。
- **隐式依赖**：工具函数依赖全局状态而非显式参数，增加了代码理解和维护难度。
- **异常安全**：如果 `set_index_context()` 和 `clear_index_context()` 之间发生异常，状态可能无法正确清理。

**重构计划**：

此 ADR 标记为"待重构"，计划在未来版本中引入 `contextvars` 或依赖注入容器来替代模块级全局变量。

### References

- `src/fs_explorer/agent.py` — set_index_context, clear_index_context 实现
- `tests/conftest.py` — reset_agent 测试辅助函数

---

## ADR-006: Docling 作为文档解析引擎

**Status**: Accepted
**Date**: 2026-01

### Context

FsExplorer 需要支持多种文档格式的解析，包括：

- **PDF**：最常见的文档格式，包括扫描版和文本版。
- **DOCX**：Microsoft Word 文档。
- **PPTX**：Microsoft PowerPoint 演示文稿。
- **XLSX**：Microsoft Excel 电子表格。
- **HTML**：网页和电子邮件。
- **Markdown**：纯文本标记格式。

选型时的核心需求：

1. **统一 API**：希望使用同一个库处理所有格式，而不是为每种格式引入不同的依赖。
2. **高质量输出**：解析结果应该是结构化的 Markdown，保留原始文档的层次结构。
3. **表格支持**：能够正确提取和转换表格数据。
4. **元数据提取**：能够提取文档的元数据（作者、创建时间等）。

### Decision

选择 **Docling** 作为统一的文档解析引擎。

```python
# fs.py
from docling.document_converter import DocumentConverter

_DOCUMENT_CACHE: dict = {}

def parse_document(file_path: str) -> str:
    """将文档解析为 Markdown 格式。"""
    converter = DocumentConverter()
    result = converter.convert(file_path)
    return result.document.export_to_markdown()
```

### Alternatives

| 方案 | 优势 | 劣势 | 未选择原因 |
|------|------|------|-----------|
| **PyMuPDF (fitz)** | PDF 解析速度快，功能丰富 | 仅支持 PDF，不支持 Office 格式 | 需要为不同格式引入多个库 |
| **python-docx + python-pptx + openpyxl** | 原生 Office 格式支持 | 每个格式需要单独处理，输出格式不统一 | 维护成本高，输出质量参差不齐 |
| **Apache Tika** | 支持格式最全面 | Java 依赖，启动慢，Python 绑定不够友好 | 部署复杂，性能不佳 |
| **Unstructured.io** | 专为 LLM 应用设计 | 商业许可，部分功能收费 | 成本问题，依赖外部服务 |
| **MarkItDown** | 轻量级，Microsoft 开发 | 功能相对简单，表格处理不如 Docling | 表格和复杂布局支持不足 |

### Consequences

**正面影响：**

- **统一 API**：一个 `DocumentConverter` 类处理所有支持的格式，代码简洁。
- **高质量 Markdown 输出**：保留标题、列表、表格等文档结构，LLM 易于理解。
- **表格支持**：能够正确提取表格并转换为 Markdown 表格格式。
- **活跃开发**：Docling 由 IBM 团队维护，社区活跃，功能持续完善。

**负面影响：**

- **重依赖**：Docling 依赖 PyTorch 等深度学习框架，安装包体积较大（约 500MB）。
- **解析速度**：对于大型 PDF 文件，解析可能需要较长时间（10-30 秒）。
- **内存占用**：解析过程中需要较多内存，大文件可能导致 OOM。
- **格式限制**：不支持旧版 Office 格式（`.doc`、`.xls`）和某些特殊格式。

### References

- `src/fs_explorer/fs.py` — parse_document 实现
- Docling 官方文档: https://docling-project.github.io/

---

## ADR-007: 混合检索设计（语义+元数据）

**Status**: Accepted
**Date**: 2026-01

### Context

FsExplorer 的索引模式需要支持多种检索场景：

1. **语义检索**：用户查询"收购价格"，需要找到包含"purchase price"、"acquisition cost" 等相关表述的文档。
2. **元数据检索**：用户查询"2024 年的合同"，需要过滤 `document_type=contract` 且 `year=2024` 的文档。
3. **组合查询**：用户查询"2024 年关于收购的合同"，需要同时满足语义相关性和元数据过滤条件。

单一检索方式无法覆盖所有场景，需要设计一个混合检索系统。

### Decision

设计**并行混合检索**架构，同时执行语义检索和元数据过滤，然后合并排序结果。

```python
# search/query.py
class IndexedQueryEngine:
    def search(self, query: str, filters: Optional[List[MetadataFilter]] = None, limit: int = 10) -> List[RankedDocument]:
        with ThreadPoolExecutor(max_workers=2) as executor:
            # 并行执行两条检索路径
            semantic_future = executor.submit(self._semantic_search, query, limit * 2)
            metadata_future = executor.submit(self._metadata_search, filters, limit * 2) if filters else None
            
            semantic_results = semantic_future.result()
            metadata_results = metadata_future.result() if metadata_future else []
        
        # 合并排序
        return rank_documents(semantic_results, metadata_results, limit)
```

评分公式：

```python
# search/ranker.py
@dataclass
class RankedDocument:
    doc_id: str
    semantic_score: float  # 0.0 - 1.0
    metadata_score: float  # 0.0 - 1.0
    
    @property
    def combined_score(self) -> float:
        return self.semantic_score * 100 + self.metadata_score * 10
```

### Alternatives

| 方案 | 优势 | 劣势 | 未选择原因 |
|------|------|------|-----------|
| **仅语义检索** | 实现简单 | 无法利用结构化元数据 | 过滤能力不足 |
| **仅元数据检索** | 精确过滤 | 无法理解语义相关性 | 召回率差 |
| **串行执行** | 可以用第一路结果优化第二路查询 | 延迟增加，无法并行优化 | 性能不如并行方案 |
| **RRF 融合** | 经典的排序融合算法 | 需要调优参数 | 当前评分公式已足够有效 |

### Consequences

**正面影响：**

- **召回率提升**：两条检索路径互补，减少遗漏相关文档的概率。
- **灵活性**：用户可以单独使用语义检索、单独使用元数据过滤，或组合使用。
- **并行执行**：利用 ThreadPoolExecutor 并行处理，总延迟取决于较慢的一路。
- **可扩展**：未来可以轻松添加第三条检索路径（如基于知识图谱的检索）。

**负面影响：**

- **实现复杂度**：需要处理两条路径的结果合并、去重、排序。
- **延迟增加**：即使只需要语义检索，元数据检索也会消耗资源（虽然并行执行减少了影响）。
- **评分调优**：`combined_score` 的权重（100 vs 10）需要根据实际场景调优。
- **一致性挑战**：两路结果可能不一致（语义认为相关但元数据不匹配），需要合理的冲突解决策略。

### References

- `src/fs_explorer/search/query.py` — IndexedQueryEngine 实现
- `src/fs_explorer/search/ranker.py` — RankedDocument 和 rank_documents

---

## ADR-008: 元数据过滤 DSL

**Status**: Accepted
**Date**: 2026-01

### Context

FsExplorer 的 Agent 需要构造元数据查询条件来过滤文档。例如：

- 查找所有 PDF 文档：`extension = .pdf`
- 查找包含货币信息的文档：`mentions_currency = true`
- 查找特定年份的文档：`year >= 2020`
- 查找包含特定关键词的文档：`filename ~ contract`

设计一个 DSL（领域特定语言）需要平衡：

1. **可读性**：DSL 应该易于人类阅读和理解。
2. **LLM 友好**：LLM 应该能够根据自然语言查询生成正确的 DSL 表达式。
3. **表达力**：DSL 应该支持常见的比较操作（等于、大于、包含等）。
4. **可解析性**：DSL 应该易于解析，避免歧义。

### Decision

设计**类 SQL 的 DSL**，支持以下操作符：

```python
# search/filters.py
def parse_metadata_filters(filter_string: str) -> List[MetadataFilter]:
    """
    解析过滤 DSL 字符串为 MetadataFilter 对象列表。
    
    支持的语法：
    - field=value       精确匹配（字符串）
    - field=true/false  布尔匹配
    - field>=num        大于等于（数字）
    - field<=num        小于等于（数字）
    - field>num         大于（数字）
    - field<num         小于（数字）
    - field in (a,b,c)  枚举匹配
    - field~substring   子字符串匹配（LIKE）
    
    多个条件用逗号分隔：
    - extension=pdf, year>=2020, filename~contract
    """
    ...
```

使用示例：

```bash
# CLI 使用
uv run explore query --task "Find contracts" --filter "extension=pdf, year>=2020"

# Agent 内部使用
filters = parse_metadata_filters("document_type=contract, mentions_currency=true")
results = query_engine.search("purchase price", filters=filters)
```

### Alternatives

| 方案 | 优势 | 劣势 | 未选择原因 |
|------|------|------|-----------|
| **纯 JSON** | 无解析开销，程序友好 | 可读性差，LLM 生成容易出错 | 不适合人类阅读和 LLM 生成 |
| **SQL WHERE 子句** | 表达力最强，标准熟悉 | 解析复杂，SQL 注入风险 | 过于重量级，安全考量 |
| **自然语言** | 最直观 | 需要额外的 NLP 解析步骤 | 实现复杂，歧义多 |
| **MongoDB 查询语法** | 表达力好，JSON 格式 | 学习曲线，解析复杂 | 对于当前需求过于复杂 |

### Consequences

**正面影响：**

- **可读性**：DSL 语法接近自然语言，人类和 LLM 都容易理解。
- **LLM 友好**：在 SYSTEM_PROMPT 中给出几个示例后，LLM 能够准确生成 DSL 表达式。
- **易于解析**：简单的正则表达式即可解析，无需复杂的语法分析器。
- **类型安全**：解析器自动处理字符串、数字、布尔值的类型转换。

**负面影响：**

- **功能有限**：不支持 AND/OR/NOT 逻辑组合，复杂查询需要多次过滤。
- **解析复杂度**：边界情况（如值中包含逗号、括号）需要特殊处理。
- **错误提示**：DSL 语法错误时的错误信息可能不够友好。
- **扩展性**：添加新操作符需要修改解析器代码。

### References

- `src/fs_explorer/search/filters.py` — parse_metadata_filters 实现
- `src/fs_explorer/agent.py` — SYSTEM_PROMPT 中的 DSL 说明

---

## ADR-009: langextract 可选集成

**Status**: Accepted
**Date**: 2026-01

### Context

FsExplorer 的索引模式需要提取丰富的元数据来支持高级过滤和检索。例如：

- **人名**：识别文档中提到的关键人物（如"John Smith"）。
- **组织名**：识别文档中提到的公司、机构（如"Acme Corp"）。
- **金额**：识别文档中的货币金额（如"$2.5 million"）。
- **日期**：识别文档中的关键日期（如"March 15, 2024"）。
- **交易条款**：识别收购协议中的关键条款（如"earnout"、"escrow"）。

纯正则表达式或启发式方法无法高质量地完成这些提取任务，需要 LLM 的语义理解能力。但强制依赖 LLM 提取会增加成本和复杂性。

### Decision

将 **langextract** 作为**可选功能**集成，提供默认 profile 和自动发现机制。

```python
# indexing/metadata.py
def extract_metadata(file_path: str, content: str, use_langextract: bool = False, profile: Optional[str] = None) -> dict:
    """提取文档元数据。"""
    metadata = _extract_heuristic_metadata(file_path, content)
    
    if use_langextract:
        langextract_data = _extract_with_langextract(content, profile or _auto_discover_profile(file_path))
        metadata.update(langextract_data)
    
    return metadata

def _auto_discover_profile(file_path: str) -> str:
    """根据文件类型自动选择合适的提取 profile。"""
    if "acquisition" in file_path.lower():
        return "deal_terms"
    elif "contract" in file_path.lower():
        return "contract_terms"
    return "general"
```

配置方式：

```bash
# 启用 langextract
uv run explore index data/test_acquisition/ --with-langextract

# 配置 langextract 模型
export FS_EXPLORER_LANGEXTRACT_MODEL="gemini-3-flash-preview"
export FS_EXPLORER_LANGEXTRACT_MAX_CHARS=6000
```

### Alternatives

| 方案 | 优势 | 劣势 | 未选择原因 |
|------|------|------|-----------|
| **纯正则表达式** | 无外部依赖，速度快 | 召回率低，维护成本高 | 无法满足高质量提取需求 |
| **自研 NER 模型** | 完全可控 | 需要训练数据和模型开发能力 | 开发成本过高 |
| **其他 LLM 库（如 spaCy + LLM）** | 功能丰富 | 集成复杂，API 成本 | 不如 langextract 专用 |
| **强制依赖 langextract** | 统一体验 | 增加上手门槛，部分用户不需要 | 违背可选设计原则 |

### Consequences

**正面影响：**

- **高质量实体提取**：利用 LLM 的语义理解能力，提取准确率远高于正则方法。
- **可定制**：用户可以根据领域需求定义自定义 profile（如"并购交易"、"法律合同"）。
- **可选设计**：不需要的用户可以跳过，降低上手门槛。
- **自动发现**：系统可以根据文件路径和内容自动选择合适的 profile。

**负面影响：**

- **API 成本**：每个文档都需要额外的 LLM 调用，增加了索引成本。
- **可选依赖管理**：需要处理 langextract 未安装时的降级逻辑。
- **配置复杂性**：用户需要理解 profile 概念和配置选项。
- **版本兼容性**：langextract 库的 API 变化可能影响集成。

### References

- `src/fs_explorer/indexing/metadata.py` — extract_metadata 实现
- langextract 文档: https://github.com/google/langextract

---

## ADR-010: SHA1 稳定 ID 策略

**Status**: Accepted
**Date**: 2026-01

### Context

FsExplorer 的索引系统需要为每个文档和 chunk 生成唯一标识符。ID 策略需要满足以下需求：

1. **幂等性**：重新索引同一文档集应该产生相同的 ID，避免重复数据。
2. **去重**：同一文档的不同路径变体（如相对路径 vs 绝对路径）应该生成相同的 ID。
3. **稳定性**：文档内容微小变化（如修改一个错别字）不应该改变 ID（除非使用内容哈希）。
4. **分布式友好**：ID 生成不应依赖全局状态（如自增序列），以便支持未来的分布式索引。

### Decision

使用 **SHA1 哈希**生成稳定的文档和 chunk ID。

```python
# storage/duckdb.py
import hashlib

def _stable_id(*parts: str) -> str:
    """生成稳定的 SHA1 哈希 ID。"""
    content = ":".join(parts)
    return hashlib.sha1(content.encode("utf-8")).hexdigest()

def make_document_id(corpus_id: str, relative_path: str) -> str:
    """生成文档 ID：SHA1(corpus_id:relative_path)"""
    return _stable_id(corpus_id, relative_path)

def make_chunk_id(doc_id: str, position: int, start: int, end: int) -> str:
    """生成 chunk ID：SHA1(doc_id:position:start:end)"""
    return _stable_id(doc_id, str(position), str(start), str(end))
```

ID 生成规则：

| 实体 | ID 公式 | 示例 |
|------|---------|------|
| Corpus | `SHA1(root_path)` | `corp_a1b2c3d4...` |
| Document | `SHA1(corpus_id:relative_path)` | `doc_e5f6g7h8...` |
| Chunk | `SHA1(doc_id:position:start:end)` | `chk_i9j0k1l2...` |
| Schema | `SHA1(corpus_id:schema_version)` | `sch_m3n4o5p6...` |

### Alternatives

| 方案 | 优势 | 劣势 | 未选择原因 |
|------|------|------|-----------|
| **自增整数 ID** | 可读性好，存储紧凑 | 依赖全局状态，分布式场景复杂 | 不满足分布式友好需求 |
| **UUID v4** | 全球唯一，无冲突 | 随机性导致无法幂等，重新索引会生成新 ID | 不满足幂等性需求 |
| **UUID v5 (基于命名空间)** | 确定性生成，基于内容 | 需要选择命名空间，不如 SHA1 直观 | 与 SHA1 方案类似，但标准化程度较低 |
| **内容哈希 (SHA256)** | 内容变化检测 | 微小变化导致 ID 变化，不利于更新 | 不满足稳定性需求 |
| **混合 ID (时间戳+随机)** | 有序性 | 无法幂等，冲突风险 | 不适合索引场景 |

### Consequences

**正面影响：**

- **幂等性**：重新索引同一文档集产生相同的 ID，UPSERT 操作不会创建重复数据。
- **去重**：相同文档无论以何种路径形式出现，都会生成相同的 ID。
- **分布式友好**：ID 生成不依赖全局状态，多个索引节点可以独立生成 ID 而不会冲突。
- **确定性**：给定相同的输入，总是生成相同的 ID，便于测试和调试。

**负面影响：**

- **ID 不可读**：SHA1 哈希对人类不友好，调试时需要额外的映射表。
- **哈希冲突理论风险**：虽然 SHA1 冲突概率极低（2^-80），但理论上仍存在风险。
- **路径敏感性**：如果文件在目录间移动（相对路径变化），会生成新的 ID，旧 ID 成为孤儿。
- **计算开销**：SHA1 计算虽然快速，但在处理数百万个 chunk 时仍有一定开销。

**缓解措施：**

- 使用 SHA1 而非 SHA256，在安全性和性能之间取得平衡（此处不需要加密级别的安全性）。
- 定期清理孤儿 chunk（通过 `cleanup_orphaned_chunks` 方法）。
- 在日志中同时记录 ID 和人类可读的路径信息。

### References

- `src/fs_explorer/storage/duckdb.py` — _stable_id, make_document_id, make_chunk_id
- `src/fs_explorer/indexing/pipeline.py` — 索引管道中的 ID 使用

---

## ADR 状态汇总

| ADR | 标题 | 状态 | 是否有后续计划 |
|-----|------|------|---------------|
| ADR-001 | 选择 Google Gemini 作为 LLM 提供商 | Accepted | 计划引入多 LLM 抽象层 |
| ADR-002 | 事件驱动 Workflow 架构 | Accepted | 持续优化 |
| ADR-003 | DuckDB 作为嵌入式存储 | Accepted | 监控 VSS 扩展可用性 |
| ADR-004 | 三阶段文档探索策略 | Accepted | 根据使用数据调优 |
| ADR-005 | 模块级全局状态管理索引上下文 | Accepted（待重构） | 计划引入 contextvars |
| ADR-006 | Docling 作为文档解析引擎 | Accepted | 监控解析性能 |
| ADR-007 | 混合检索设计（语义+元数据） | Accepted | 计划添加更多检索路径 |
| ADR-008 | 元数据过滤 DSL | Accepted | 计划支持逻辑组合 |
| ADR-009 | langextract 可选集成 | Accepted | 扩展 profile 库 |
| ADR-010 | SHA1 稳定 ID 策略 | Accepted | 无变更计划 |

---

## 如何提出新的 ADR

如果你认为某个架构决策需要记录为 ADR，请遵循以下流程：

1. **创建新文件**：在本文档末尾添加新的 ADR，编号递增。
2. **使用模板**：按照"Status / Date / Context / Decision / Alternatives / Consequences / References"格式编写。
3. **社区评审**：在 PR 中邀请至少一位核心维护者评审。
4. **更新状态**：ADR 被接受后，更新顶部的状态汇总表。

ADR 是项目知识库的重要组成部分，帮助我们理解"为什么这样做"，而不仅仅是"做了什么"。

---

☕️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../res/promotion.jpg)