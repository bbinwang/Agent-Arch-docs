# 12. 架构决策记录（Architecture Decision Records）

> **文档编号**：12/14
> **预估字数**：~8,000 字
> **ADR 数量**：6 个关键决策

---

## 12.1 ADR-001：选用 ChromaDB 作为向量数据库

### 状态
**已接受（Accepted）**

### 背景
RAG 系统需要一个向量数据库来存储和检索密集嵌入。可选方案包括：
- **ChromaDB**：Python 原生，本地部署
- **FAISS**：Meta 开发，高性能但无元数据过滤
- **Milvus**：分布式，适合大规模部署
- **Pinecone**：云服务，无需运维但数据在云端
- **Weaviate**：开源，支持混合检索

### 决策
选用 **ChromaDB** 作为默认向量数据库。

### 理由
1. **Python 原生**：与 LangChain 生态无缝集成
2. **本地部署**：数据不出本机，满足隐私需求
3. **元数据过滤**：支持按 chunk_size、label 等过滤
4. **零运维**：无需独立服务，嵌入在应用中
5. **持久化**：支持磁盘持久化

### 后果
- **正面**：简化部署，降低运维成本
- **负面**：超大规模（百万级向量）性能可能不如 Milvus
- **缓解**：通过 VectorStore 抽象基类，未来可切换实现

### 相关代码
```python
# chroma.py
from langchain_chroma import Chroma

class VectorStoreChroma(VectorStore):
    def __init__(self, persist_folder: str, config: Config):
        self._persist_folder = persist_folder
```

---

## 12.2 ADR-002：选用 SPLADE 实现稀疏检索

### 状态
**已接受（Accepted）**

### 背景
基础 RAG 仅使用密集检索，对关键词匹配效果不佳。可选方案：
- **BM25**：经典关键词检索，无语义理解
- **SPLADE**：语义级稀疏检索，学习稀疏表示
- **TF-IDF**：传统方法，效果有限

### 决策
选用 **SPLADE**（`naver/splade-cocondenser-ensembledistil`）实现稀疏检索。

### 理由
1. **语义级稀疏**：比 BM25 更能理解语义
2. **与密集检索互补**：稀疏擅长关键词，密集擅长语义
3. **可学习**：基于 BERT 的稀疏表示，可微调
4. **论文验证**：多篇论文证明其有效性

### 后果
- **正面**：检索质量显著提升
- **负面**：需要额外内存存储稀疏索引
- **缓解**：使用 scipy.sparse 压缩存储

### 相关代码
```python
# splade.py
class SparseEmbeddingsSplade:
    def __init__(self, config, splade_model_id="naver/splade-cocondenser-ensembledistil"):
        self.model = AutoModelForMaskedLM.from_pretrained(splade_model_id)
```

---

## 12.3 ADR-003：配置驱动架构（YAML + Pydantic）

### 状态
**已接受（Accepted）**

### 背景
需要一种方式让用户配置 RAG 流水线。可选方案：
- **YAML**：可读性强，支持注释
- **JSON**：机器友好，但不支持注释
- **TOML**：类似 INI，适合简单配置
- **Python 代码**：最灵活，但门槛高

### 决策
选用 **YAML** 作为配置格式，**Pydantic** 进行验证。

### 理由
1. **可读性**：YAML 最接近自然语言
2. **注释支持**：用户可添加注释说明配置意图
3. **Pydantic 验证**：强类型验证，错误信息清晰
4. **嵌套结构**：支持复杂配置层次

### 后果
- **正面**：降低使用门槛，配置错误早发现
- **负面**：YAML 缩进敏感，可能出错
- **缓解**：提供配置模板和验证错误提示

### 相关代码
```python
# config.py
class Config(BaseModel):
    cache_folder: Path
    embeddings: EmbeddingsConfig
    semantic_search: SemanticSearchConfig
    llm: Optional[LLMConfig] = None

def get_config(path: Union[str, Path]) -> Config:
    with open(path, "r") as f:
        conf_dict = yaml.safe_load(f)
    return Config(**conf_dict)
```

---

## 12.4 ADR-004：插件式解析器架构

### 状态
**已接受（Accepted）**

### 背景
需要支持多种文档格式，且未来可能添加新格式。可选方案：
- **单一解析器**：用 Unstructured 处理所有格式
- **专用解析器**：为每种格式编写专用解析器
- **插件式**：专用解析器 + 回退机制

### 决策
选用 **插件式解析器架构**，按扩展名路由到专用解析器，回退到 Unstructured。

### 理由
1. **最佳解析效果**：每种格式有最佳解析策略
2. **可扩展**：新增格式无需修改现有代码
3. **回退机制**：未知格式自动使用 Unstructured
4. **配置灵活**：用户可为每种格式配置不同参数

### 后果
- **正面**：解析质量高，扩展性强
- **负面**：代码复杂度增加
- **缓解**：清晰的接口定义和文档

### 相关代码
```python
# parsers/splitter.py
class DocumentSplitter:
    def __init__(self, config: Config):
        self._splitter_conf = {
            "md": markdown_splitter,
            "docx": docx_splitter,
            "doc": docx_splitter,
            "pdf": PDFSplitter(chunk_overlap=200).split_document,
        }
        self._fallback_splitter = UnstructuredSplitter().split_document
```

---

## 12.5 ADR-005：MCP 协议集成

### 状态
**已接受（Accepted）**

### 背景
需要让 RAG 系统能与 AI 工具链（Cursor、Windsurf、VSCode Copilot）集成。可选方案：
- **REST API**：通用但需要客户端适配
- **MCP 协议**：新兴标准，AI 工具原生支持
- **CLI 包装**：简单但功能受限

### 决策
选用 **MCP 协议**（通过 FastApiMCP）暴露 RAG 工具。

### 理由
1. **标准化**：MCP 是 Anthropic 推出的开放标准
2. **原生集成**：Cursor/Windsurf 等原生支持
3. **工具暴露**：可将 RAG 操作暴露为 LLM 工具
4. **未来趋势**：更多 AI 工具将支持 MCP

### 后果
- **正面**：与 AI 工具链无缝集成
- **负面**：依赖 FastApiMCP 库
- **缓解**：MCP 是开放标准，可实现自己的 MCP 服务器

### 相关代码
```python
# api.py
from fastapi_mcp import FastApiMCP

mcp = FastApiMCP(
    api_app,
    name="pyLLMSearch MCP Server",
    include_operations=["rag_retrieve_chunks", "rag_generate_answer", ...],
)
mcp.mount()
```

---

## 12.6 ADR-006：LangChain 框架依赖

### 状态
**已接受（Accepted）**

### 背景
需要 LLM 编排框架来构建 RAG 流水线。可选方案：
- **原生 API 调用**：直接调用 OpenAI/HuggingFace API
- **LangChain**：最流行的 LLM 编排框架
- **LlamaIndex**：专注 RAG 的框架
- **Haystack**：企业级 RAG 框架

### 决策
选用 **LangChain** 作为 LLM 编排框架。

### 理由
1. **生态丰富**：支持多种 LLM、向量数据库、工具
2. **Chain 抽象**：将 RAG 流程抽象为可组合的 Chain
3. **社区活跃**：文档丰富，问题容易解决
4. **标准化接口**：LLM、Document、Retriever 等有标准接口

### 后果
- **正面**：开发效率高，生态集成好
- **负面**：框架升级可能带来破坏性变更
- **缓解**：锁定版本，抽象层隔离变化

### 相关代码
```python
# utils.py
from langchain_classic.chains.combine_documents import create_stuff_documents_chain

chain = create_stuff_documents_chain(llm=llm.model, prompt=llm.prompt)
```

---

## 12.7 ADR 总结

| ADR | 决策 | 状态 | 影响 |
|-----|------|------|------|
| ADR-001 | ChromaDB 作为向量数据库 | 已接受 | 存储层 |
| ADR-002 | SPLADE 实现稀疏检索 | 已接受 | 检索层 |
| ADR-003 | YAML + Pydantic 配置 | 已接受 | 配置层 |
| ADR-004 | 插件式解析器 | 已接受 | 解析层 |
| ADR-005 | MCP 协议集成 | 已接受 | 接口层 |
| ADR-006 | LangChain 框架 | 已接受 | 编排层 |

---

## 12.8 本章小结

本章记录了 pyLLMSearch 的 6 个关键架构决策：

1. **ChromaDB**：本地部署、Python 原生、元数据过滤
2. **SPLADE**：语义级稀疏检索，与密集检索互补
3. **YAML + Pydantic**：可读性强，验证清晰
4. **插件式解析器**：最佳解析效果，可扩展
5. **MCP 协议**：与 AI 工具链集成
6. **LangChain**：生态丰富，Chain 抽象

下一章将分析核心算法。

---

☕️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../res/promotion.jpg)