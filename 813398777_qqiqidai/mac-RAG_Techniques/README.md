# Advanced RAG Techniques - 技术架构文档

> **项目**: Advanced RAG Techniques (NirDiamant/RAG_Techniques)  
> **文档版本**: v1.0  
> **生成日期**: 2026-07-26  
> **文档作者**: 系统架构分析  
> **文档状态**: 完整版 ✅

---

## 文档导航

本文档是对 **Advanced RAG Techniques** 项目的完整技术架构分析，包含 42+ 个 RAG 技术的深度代码走读、架构设计、流程分析、数据模型、API 设计、部署方案及改进建议。

### 📊 文档统计

| 指标 | 数值 |
|------|------|
| **章节数量** | 10 章 + README |
| **总行数** | 6,287 行 |
| **总字数** | ~201 KB |
| **Mermaid 图表** | 30+ |
| **代码示例** | 50+ |

### 📁 文件列表

| 文件 | 章节 | 行数 | 大小 | 内容概要 |
|------|------|------|------|----------|
| [01-project-overview.md](./01-project-overview.md) | 项目概述 | 378 | 17 KB | 目标、技术栈、架构风格、功能特性、非功能性需求 |
| [02-c4-architecture.md](./02-c4-architecture.md) | C4 架构模型 | 412 | 21 KB | Context/Container/Component/Code 四层 Mermaid 图及详解 |
| [03-flows-sequence.md](./03-flows-sequence.md) | 系统流程与时序图 | 967 | 29 KB | 12+ 核心业务流程 Mermaid 流程图 + 时序图详解 |
| [04-module-structure.md](./04-module-structure.md) | 模块/包结构与依赖分析 | 491 | 19 KB | 目录树、模块职责、依赖关系图 |
| [05-core-code-walkthrough.md](./05-core-code-walkthrough.md) | 核心代码讲解 | 1,140 | 39 KB | 逐文件、逐函数深度走读（最重要章节） |
| [06-data-model.md](./06-data-model.md) | 数据模型与数据库设计 | 508 | 13 KB | ER 图、向量存储结构、缓存策略 |
| [07-api-design.md](./07-api-design.md) | API 与接口设计 | 511 | 13 KB | 内部 API、LLM 接口、数据流接口 |
| [08-deployment.md](./08-deployment.md) | 部署、运维与基础设施 | 607 | 14 KB | CI/CD、Docker、K8s、监控方案 |
| [09-improvements.md](./09-improvements.md) | 改进建议与未来规划 | 430 | 12 KB | 优缺点、技术债、优化建议 |
| [10-developer-guide.md](./10-developer-guide.md) | 开发者上手指南 | 774 | 20 KB | 本地运行、调试、测试、ADR、算法分析 |

---

## 项目快速概览

**Advanced RAG Techniques** 是一个社区驱动的开源项目，包含 **42+ 个可运行的 Jupyter Notebook 和 Python 脚本**，覆盖从基础到前沿的 RAG（Retrieval-Augmented Generation，检索增强生成）技术。

### 技术分类体系

| 类别 | 技术数量 | 代表技术 |
|------|----------|----------|
| **Foundation 基础** | 5 | Simple RAG, CSV RAG, Reliable RAG, Chunk Size, Proposition |
| **Query Enhancement 查询增强** | 3 | Query Transformations, HyDE, HyPE |
| **Context Enrichment 上下文丰富** | 6 | Chunk Headers, RSE, Context Window, Semantic Chunking, Compression, Augmentation |
| **Advanced Retrieval 高级检索** | 5 | Fusion, Reranking, Filtering, Hierarchical, Dartboard |
| **Multi-modal 多模态** | 2 | Captioning RAG, Colpali RAG |
| **Iterative/Adaptive 迭代自适应** | 3 | Feedback Loop, Adaptive RAG, Self-RAG |
| **Graph-based 图增强** | 5 | Graph RAG, Microsoft GraphRAG, Milvus GraphRAG, RAPTOR, Local GraphRAG |
| **Evaluation 评估** | 5 | DeepEval, GroUSE, End-to-End, Open-RAG-Eval, Define Metrics |
| **Agentic/Self-Reflective** | 3 | Agentic RAG, CRAG, MemoRAG |
| **Explainability 可解释性** | 1 | Explainable Retrieval |

---

## 文档内容覆盖矩阵

| 要求章节 | 对应文件 | 完成状态 |
|----------|----------|----------|
| 1. 项目概述 | 01-project-overview.md | ✅ |
| 2. C4 架构模型（4 层 + Mermaid） | 02-c4-architecture.md | ✅ |
| 3. 系统流程与时序图（6-10+） | 03-flows-sequence.md | ✅ |
| 4. 模块/包结构与依赖分析 | 04-module-structure.md | ✅ |
| 5. 核心代码讲解（逐文件、逐函数） | 05-core-code-walkthrough.md | ✅ |
| 6. 数据模型与数据库设计 | 06-data-model.md | ✅ |
| 7. API 与接口设计 | 07-api-design.md | ✅ |
| 8. 部署、运维与基础设施 | 08-deployment.md | ✅ |
| 9. 改进建议、风险点与未来规划 | 09-improvements.md | ✅ |
| 10. 额外增强内容 | 10-developer-guide.md | ✅ |

### 额外增强内容覆盖

| 增强内容 | 对应位置 | 完成状态 |
|----------|----------|----------|
| 代码走读文档（Code Walkthrough） | 05-core-code-walkthrough.md | ✅ |
| 开发者上手指南 | 10-developer-guide.md | ✅ |
| 架构决策记录（ADR） | 10-developer-guide.md §10.3 | ✅ |
| 关键算法伪代码 + 复杂度分析 | 10-developer-guide.md §10.4 | ✅ |
| 测试策略和主要测试用例 | 10-developer-guide.md §10.5 | ✅ |

---

## Mermaid 图表清单

| 图表类型 | 数量 | 分布 |
|----------|------|------|
| **C4 Context** | 1 | 02-c4-architecture.md |
| **C4 Container** | 1 | 02-c4-architecture.md |
| **C4 Component** | 1 | 02-c4-architecture.md |
| **C4 Deployment** | 1 | 02-c4-architecture.md |
| **Flowchart** | 15+ | 03, 06, 07, 09 |
| **Sequence Diagram** | 10+ | 03-flows-sequence.md |
| **ER Diagram** | 1 | 06-data-model.md |
| **State Diagram** | 1 | 06-data-model.md |
| **Gantt** | 1 | 09-improvements.md |

---

## 阅读指南

| 读者角色 | 推荐阅读顺序 | 预计时间 |
|----------|-------------|----------|
| **架构师** | 01 → 02 → 03 → 09 | 2 小时 |
| **开发者** | 01 → 04 → 05 → 10 | 3 小时 |
| **运维/DevOps** | 01 → 08 → 09 | 1.5 小时 |
| **研究者** | 01 → 03 → 06 → 07 | 2 小时 |
| **评估/QA** | 01 → 05 → 06 → 08 | 2 小时 |

---

## 文档约定

- 全文使用 **中文**，语言专业、严谨、细节丰富
- 所有图表使用标准 **Mermaid** 代码块（可直接渲染）
- 每个图表搭配 300-500 字详细文字说明
- 代码引用格式：`文件路径:行号`
- 术语首次出现时附英文原文
- 所有分析严格基于项目实际代码，杜绝虚构
- 内容越详细越好，禁止简略或省略

---

## 项目核心统计

| 指标 | 数值 |
|------|------|
| Notebook 数量 | 40+ |
| Runnable Script 数量 | 20 |
| 技术类别 | 10 |
| 支持的语言框架 | LangChain, LlamaIndex |
| 支持的 LLM | OpenAI, Anthropic, Groq, Bedrock, Ollama |
| 支持的向量存储 | FAISS, Milvus, ChromaDB, Neo4j |
| 支持的评估框架 | DeepEval, GroUSE, RAGAS, Open-RAG-Eval |
| 测试文件 | 2 (conftest.py, test_imports.py) |
| CI 工作流 | 2 (github-test.yml, local-test.yml) |
| 数据文件 | 7 (PDF, CSV, JSON, TXT) |
| 核心共享模块 | 1 (helper_functions.py, 362 行) |

---

## 快速开始

```bash
# 克隆仓库
git clone https://github.com/NirDiamant/RAG_Techniques.git
cd RAG_Techniques

# 创建虚拟环境
python -m venv venv
source venv/bin/activate

# 安装依赖
pip install -r requirements.txt

# 配置 API Key
echo "OPENAI_API_KEY=your-key" > .env

# 运行测试
pytest

# 运行 Simple RAG
python all_rag_techniques_runnable_scripts/simple_rag.py \
    --path data/Understanding_Climate_Change.pdf \
    --query "What is climate change?"
```

---

## 许可证

本文档与原始项目保持相同许可证。

---

## 反馈与贡献

如发现文档错误或改进建议，欢迎提交 Issue 或 PR。
