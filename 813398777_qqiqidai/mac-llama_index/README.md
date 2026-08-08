# LlamaIndex 项目技术架构文档

> **版本**: v0.14.23
> **生成日期**: 2026-07-26
> **分析范围**: 全量源码（llama-index-core 480+ 文件 / ~75K 行 + 31 类集成包 / 600+ 子包）
> **文档性质**: 专业级项目设计与代码讲解文档

---

## 文档导航

本文档是对 LlamaIndex 开源项目的完整技术架构分析，共 15 章，拆分为 16 个 Markdown 文件。

| 文件 | 章节 | 内容概要 |
|------|------|----------|
| [01-project-overview.md](./01-project-overview.md) | 项目概述 | 目标、技术栈、架构风格、功能特性、非功能性需求 |
| [02-c4-architecture.md](./02-c4-architecture.md) | C4 架构模型 | Context/Container/Component/Code 四层 Mermaid 图 |
| [03-flows-sequence.md](./03-flows-sequence.md) | 系统流程与时序图 | 10+ 核心业务流程 Mermaid 流程图/时序图 |
| [04-module-structure.md](./04-module-structure.md) | 模块/包结构与依赖分析 | 目录树、模块职责、依赖关系图 |
| [05-code-walkthrough.md](./05-code-walkthrough.md) | 核心代码讲解（上） | LLM/Embedding/Index/Retriever/QueryEngine |
| [06-code-walkthrough-2.md](./06-code-walkthrough-2.md) | 核心代码讲解（下） | Ingestion/Workflow/Agent/Storage/Callback/Prompt/Schema |
| [07-data-model.md](./07-data-model.md) | 数据模型与数据库设计 | ER 图、表结构、缓存策略、序列化 |
| [08-api-design.md](./08-api-design.md) | API 与接口设计 | 核心 API 列表、请求/响应、Settings |
| [09-deployment-ops.md](./09-deployment-ops.md) | 部署、运维与基础设施 | 部署架构、CI/CD、工具链、发布流程 |
| [10-improvement-risks.md](./10-improvement-risks.md) | 改进建议、风险点与未来规划 | 优缺点、技术债、优化建议 |
| [11-code-walkthrough-per-component.md](./11-code-walkthrough-per-component.md) | 组件独立代码走读 | 7 大核心组件独立走读 |
| [12-developer-guide.md](./12-developer-guide.md) | 开发者上手指南 | 环境、运行、测试、调试、贡献 |
| [13-adr.md](./13-adr.md) | 架构决策记录 | 10 份 ADR |
| [14-algorithms-complexity.md](./14-algorithms-complexity.md) | 关键算法与复杂度分析 | 10 个核心算法伪代码 + 复杂度 |
| [15-testing-strategy.md](./15-testing-strategy.md) | 测试策略与主要测试用例 | 测试架构、用例、覆盖率 |

---

## 阅读路径建议

| 读者角色 | 推荐路径 |
|----------|----------|
| **快速了解项目** | 01 → 02 → 03 |
| **架构师 / 技术评审** | 02 → 04 → 07 → 10 → 13 |
| **开发者（上手）** | 01 → 12 → 05 → 06 → 08 |
| **代码贡献者** | 12 → 04 → 05 → 06 → 15 |
| **运维 / SRE** | 09 → 02 → 07 |
| **算法 / R&D** | 14 → 05 → 06 → 07 |

---

## 术语表

| 术语 | 含义 |
|------|------|
| **RAG** | Retrieval-Augmented Generation，检索增强生成 |
| **LLM** | Large Language Model，大语言模型 |
| **Embedding** | 文本向量嵌入 |
| **Node** | 文档切片后的最小索引单元（TextNode） |
| **Index** | 索引结构（Vector/Tree/KG/Keyword/PropertyGraph 等） |
| **Retriever** | 检索器，从索引中召回相关 Node |
| **QueryEngine** | 查询引擎，组合 Retriever + Synthesizer |
| **Synthesizer** | 响应合成器，将召回结果 + Query 送入 LLM 生成回答 |
| **Agent** | 智能体，支持多步推理 + 工具调用 |
| **Workflow** | 事件驱动的工作流引擎（基于 llama-index-workflows） |
| **Ingestion** | 数据摄入（解析 → 转换 → 存储） |
| **DocStore** | 文档存储 |
| **IndexStore** | 索引结构存储 |
| **VectorStore** | 向量存储 |
| **GraphStore** | 图存储 |
| **Callback** | 回调机制（旧事件系统） |
| **Instrumentation** | 可观测性系统（新事件/Span 系统） |
| **PromptMixin** | Prompt 管理混入类 |
| **TransformComponent** | 转换组件基类 |
| **QueryBundle** | 查询封装对象 |
| **NodeWithScore** | 带评分的节点 |
| **IndexNode** | 索引节点（可引用其他对象） |
| **RefDoc** | 引用文档（原始文档引用信息） |
| **ContentBlock** | 内容块（Text/Image/Audio/Video/Document/Thinking） |
| **StartEvent/StopEvent** | Workflow 起止事件 |
| **Context** | Workflow 上下文（状态共享） |
| **Dispatcher** | 事件派发器（Instrumentation 核心） |
| **Span** | 调用链 Span |

---

## 缩略语表

| 缩略语 | 全称 |
|--------|------|
| **ABC** | Abstract Base Class |
| **API** | Application Programming Interface |
| **ADR** | Architecture Decision Record |
| **CI/CD** | Continuous Integration / Continuous Delivery |
| **ER** | Entity-Relationship |
| **KV** | Key-Value |
| **LPG** | Labeled Property Graph |
| **MCP** | Model Context Protocol |
| **Mermaid** | 文本图表描述语言 |
| **NL** | Natural Language |
| **OCR** | Optical Character Recognition |
| **PG** | Property Graph |
| **Pydantic** | Python 数据校验库 |
| **RAKE** | Rapid Automatic Keyword Extraction |
| **SQL** | Structured Query Language |
| **UV** | 现代 Python 包管理器（Astral 出品） |

---

## 文档约定

- **文件路径**: 全部使用相对于仓库根目录的路径，如 `llama-index-core/llama_index/core/llms/llm.py`
- **代码引用**: 标注文件路径 + 行号范围
- **图表**: 全部使用标准 Mermaid 代码块，可在 GitHub / Typora / Obsidian 中直接渲染
- **类/方法**: 使用 `ClassName.method_name()` 格式
- **重要提示**: 使用 `> **Note**` 引用块
- **警告/风险**: 使用 `> **Warning**` 引用块
