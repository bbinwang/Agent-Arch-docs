# 🧠 可控 RAG Agent 项目 - 技术架构文档总目录

> **文档版本**: v1.0  
> **生成日期**: 2026-07-26  
> **文档作者**: 架构文档专家 (AI)  
> **目标读者**: 系统架构师、后端工程师、AI 工程师、技术管理者  
> **文档性质**: 项目设计文档 + 代码讲解文档 + 开发者指南 + ADR

---

## 📖 文档导读

本文档是对 [Controllable-RAG-Agent](https://github.com/NirDiamant/Controllable-RAG-Agent) 项目的**全方位、极致深度**技术分析。该项目是一个基于 LangGraph 的**可控自主智能体（Controllable Autonomous Agent）**，通过确定性图（Deterministic Graph）作为"大脑"，解决传统基于语义相似度的 RAG 无法处理的复杂多跳推理问题。

### 文档结构总览

| 编号 | 文件 | 章节标题 | 核心内容 | 预计篇幅 |
|------|------|----------|----------|----------|
| 01 | [01-project-overview.md](./01-project-overview.md) | 项目概述 | 目标、技术栈、架构风格、功能/非功能需求 | ~8K 字 |
| 02 | [02-c4-architecture.md](./02-c4-architecture.md) | C4 架构模型 | Context/Container/Component/Code 四层 + Mermaid 图 | ~12K 字 |
| 03 | [03-flows-and-sequence.md](./03-flows-and-sequence.md) | 系统流程与时序图 | 10+ 核心业务流程图 + 时序图 | ~15K 字 |
| 04 | [04-module-structure.md](./04-module-structure.md) | 模块/包结构与依赖分析 | 目录树、模块职责、依赖图 | ~8K 字 |
| 05 | [05-core-code-walkthrough.md](./05-core-code-walkthrough.md) | 核心代码讲解 | 逐文件、逐函数深度走读 | ~20K 字 |
| 06 | [06-data-model.md](./06-data-model.md) | 数据模型与数据库设计 | FAISS 向量存储结构、数据流向、缓存策略 | ~6K 字 |
| 07 | [07-api-design.md](./07-api-design.md) | API 与接口设计 | 内部 API、LLM 调用接口、Streamlit 接口 | ~6K 字 |
| 08 | [08-deployment.md](./08-deployment.md) | 部署、运维与基础设施 | Docker、CI/CD、监控、日志 | ~6K 字 |
| 09 | [09-improvements.md](./09-improvements.md) | 改进建议、风险点与未来规划 | 优缺点、技术债、优化建议 | ~8K 字 |
| 10 | [10-developer-guide.md](./10-developer-guide.md) | 开发者上手指南 | 本地运行、调试、测试流程 | ~5K 字 |
| 11 | [11-adr.md](./11-adr.md) | 架构决策记录 (ADR) | 关键决策的历史与理由 | ~6K 字 |
| 12 | [12-algorithms.md](./12-algorithms.md) | 算法与复杂度分析 | 核心算法伪代码、时间/空间复杂度 | ~6K 字 |
| 13 | [13-testing.md](./13-testing.md) | 测试策略与用例 | 测试金字塔、Ragas 评估、用例设计 | ~5K 字 |

**总预计篇幅**: ~110K+ 字（含代码、图表、Mermaid 渲染）

---

## 🎯 快速导航

### 按读者角色推荐路径

| 角色 | 推荐阅读顺序 | 目的 |
|------|-------------|------|
| **技术管理者 / CTO** | 01 → 02 → 09 → 11 | 快速理解项目价值、架构全貌、风险与决策 |
| **系统架构师** | 01 → 02 → 03 → 04 → 06 → 09 | 深入理解架构设计、数据模型、部署方案 |
| **AI / LLM 工程师** | 01 → 03 → 05 → 12 → 13 | 掌握 Agent 核心逻辑、算法、评估方法 |
| **后端工程师** | 01 → 04 → 05 → 07 → 08 → 10 | 快速上手开发、部署、API 集成 |
| **新贡献者** | 10 → 01 → 04 → 05 → 11 | 从零开始理解项目 |

### 按主题推荐路径

| 主题 | 相关章节 |
|------|----------|
| **LangGraph 图编排** | 02 (Component/Code) → 03 (流程图) → 05 (代码走读) |
| **RAG 检索策略** | 01 → 03 (检索流程) → 06 (向量存储) → 12 (算法) |
| **幻觉检测与防控** | 03 (验证流程) → 05 (is_grounded 函数) → 11 (ADR-006) |
| **计划与执行 (Plan & Execute)** | 02 → 03 (主循环) → 05 (create_agent) → 12 |
| **匿名化与去匿名化** | 03 → 05 → 11 (ADR-004) |
| **评估与质量保障** | 06 → 13 → 09 |

---

## 🔑 核心概念速查

| 概念 | 简要说明 | 详细参考 |
|------|----------|----------|
| **Plan-and-Execute** | 将复杂问题分解为多步计划，逐步执行并动态重规划 | ADR-001, 03, 05 |
| **Anonymization** | 将问题中的命名实体替换为变量，消除 LLM 先验偏见 | ADR-004, 05 |
| **Qualitative Retrieval** | 检索 + 蒸馏 + 验证的三段式检索子图 | 02, 03, 05 |
| **Hallucination Check** | 通过 LLM 判断答案是否基于上下文，防止幻觉 | 05, 11 |
| **Chain-of-Thought (CoT)** | 在 prompt 中嵌入推理示例，引导逐步推理 | 05, 12 |
| **FAISS Vector Store** | Facebook 的向量相似度检索库，存储文档嵌入 | 06 |
| **Ragas** | RAG 评估框架，提供多维质量指标 | 06, 13 |
| **LangGraph** | 基于状态图的大模型编排框架 | 02, 04, 05 |

---

## 📊 项目关键指标

| 指标 | 数值 |
|------|------|
| 核心 Python 文件数 | 3 个（+ 2 个 Notebook） |
| 代码总行数 | ~1,868 行（不含 Notebook） |
| LangGraph 图节点数 | 11 个（主图）+ 4 个子图 |
| LLM 调用链数 | 12 条独立 Chain |
| 向量存储数量 | 3 个（chunks / summaries / quotes） |
| 支持 LLM 后端 | OpenAI GPT-4o / GPT-3.5 / Groq (Llama3) |
| 部署方式 | Docker / Docker Compose / 本地 Streamlit |
| 评估维度 | 5 项 Ragas 指标 |

---

## 📝 文档约定

### 图表约定
- 所有图表使用 **Mermaid** 语法，支持 GitHub / GitLab / Notion 等平台直接渲染
- 流程图使用 `flowchart TD/LR`
- 时序图使用 `sequenceDiagram`
- 类图使用 `classDiagram`
- 状态图使用 `stateDiagram-v2`

### 代码约定
- 代码块标注语言类型（`python`, `bash`, `yaml`, `dockerfile`）
- 关键行内联注释使用 `# ▸` 标记
- 函数签名使用 `func_name(param: type) -> return_type` 格式

### 引用约定
- 文件引用：`path/to/file.py:行号`
- 函数引用：`file.py::function_name()`
- ADR 引用：`ADR-00X`

---

## 🔄 文档更新记录

| 版本 | 日期 | 变更内容 | 作者 |
|------|------|----------|------|
| v1.0 | 2026-07-26 | 初始版本，完整 13 章 | AI 架构文档专家 |

---

> **下一步**: 请从 [01-project-overview.md](./01-project-overview.md) 开始阅读，或根据上方导航表跳转到您感兴趣的章节。
