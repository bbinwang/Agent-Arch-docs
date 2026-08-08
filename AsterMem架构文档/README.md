# AsterMem 架构文档

> **自托管个人记忆服务** — 为你和你的 AI Agent 打造的本地记忆系统

**文档版本**: 1.0 · **分析日期**: 2026-07-29 · **项目版本**: v2.0.0 · **许可证**: AGPL-3.0

**GitHub**: [Asterove/AsterMem](https://github.com/Asterove/AsterMem) · **官网**: [asterove.com](https://asterove.com/) · **作者**: [@Asterove_ai](https://x.com/Asterove_ai)

---

## 文档导航

| 文档 | 内容 | 适合谁读 |
|------|------|----------|
| [01-overview.md](01-overview.md) | 项目概述、技术栈、核心特性、架构理念 | 所有人（先读这个） |
| [02-c4-architecture.md](02-c4-architecture.md) | C4 四层架构模型（Context → Container → Component → Code）+ Mermaid 图 | 架构师、技术负责人 |
| [03-flows.md](03-flows.md) | 10 大核心业务流程 + 时序图 | 后端工程师、AI Agent 开发者 |
| [04-modules.md](04-modules.md) | 完整目录树、模块职责、依赖关系图 | 全栈工程师 |
| [05-code-walkthrough.md](05-code-walkthrough.md) | 核心代码逐文件、逐函数走读 | 深入理解代码的贡献者 |
| [06-data-model.md](06-data-model.md) | 数据模型、ER 图、SQLite 表结构、向量索引 | DBA、数据工程师 |
| [07-api.md](07-api.md) | ~115 个 REST API 端点、认证体系、SKILL 集成 | API 消费者、Agent 开发者 |
| [08-deployment.md](08-deployment.md) | 部署方式（Docker/裸机/Tauri 桌面）、CI/CD、运维 | DevOps、自托管用户 |
| [09-improvements.md](09-improvements.md) | 架构优缺点、技术债、安全风险、改进建议 | 维护者、架构评审 |

## 阅读建议

**快速了解项目** → 读 `01-overview.md` 的前 3 节即可（~10 分钟）

**准备部署** → 读 `01` → `08-deployment.md`

**准备二次开发** → 读 `01` → `02` → `04` → `05`

**理解 AI Agent 集成** → 读 `01` → `03-flows.md`（重点看流程 2/3/8）→ `07-api.md`

**深入数据层** → 读 `06-data-model.md`

## 术语表

| 术语 | 含义 |
|------|------|
| Memory | 一条记忆，对应一个 Markdown 文件 + SQLite 行 + 向量索引 |
| Trunk | 记忆的段落级分片（chunk），用于段落级精确检索 |
| Profile Layer | 用户画像层，将记忆库蒸馏成高密度上下文 |
| Dream | Profile 的"慢循环"LLM 整理过程，产出候选 claim |
| Agent Call | `/api/agent/call` 端点，AI Agent 的统一操作入口 |
| SKILL | Cursor/Claude Code 的技能包，封装 Agent CLI 调用 |
| Hybrid Search | 关键词搜索（Whoosh + jieba）与语义搜索（Chroma）的混合检索 |
| Adaptive Recall | 自适应召回策略：以最佳匹配为锚点的相对截断，而非固定阈值 |
| RRF | Reciprocal Rank Fusion，倒数排名融合，用于合并关键词与语义搜索结果 |
| Meta Tags | AI 从 Trunk 中自动抽取的隐式标签，增强搜索召回 |
| Provider | 模型提供商（Embedding / Chat），通过配置驱动，支持 24 种开箱即用 |

---

*本文档基于 AsterMem v2.0.0 源码深度分析生成，所有内容均来自实际代码阅读，无臆造。*
