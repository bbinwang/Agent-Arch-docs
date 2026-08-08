# Pi Agent Harness — 完整设计文档与代码讲解

> **项目**：Pi Agent Harness（`earendil-works/pi`）
> **版本**：v0.80.10（基于本地代码快照，2026-07-24）
> **作者**：Mario Zechner（Earendil Works）
> **许可**：MIT
> **文档生成方式**：基于源码逐文件走读，杜绝 hallucination

---

## 文档索引

| 文件 | 内容 | 预计字数 |
|------|------|---------|
| [01-project-overview.md](./01-project-overview.md) | 项目概述（目标、技术栈、架构风格、功能特性） | ~10,000 |
| [02-c4-architecture.md](./02-c4-architecture.md) | C4 架构模型（Context/Container/Component/Code） | ~25,000 |
| [03-system-flows.md](./03-system-flows.md) | 系统流程与时序图（10 个核心流程） | ~30,000 |
| [04-module-structure.md](./04-module-structure.md) | 模块/包结构与依赖分析 | ~15,000 |
| [05-code-walkthrough.md](./05-code-walkthrough.md) | 核心代码讲解（逐文件走读） | ~50,000 |
| [06-data-models.md](./06-data-models.md) | 数据模型与数据库设计 | ~12,000 |
| [07-api-design.md](./07-api-design.md) | API 与接口设计 | ~12,000 |
| [08-deployment.md](./08-deployment.md) | 部署、运维与基础设施 | ~10,000 |
| [09-improvements.md](./09-improvements.md) | 改进建议、风险点与未来规划 | ~10,000 |
| [10-appendix.md](./10-appendix.md) | 额外增强内容（走读/指南/ADR/算法/测试/术语表） | ~25,000 |

---

## 快速导航

- **架构总览**：[02-c4-architecture.md](./02-c4-architecture.md)
- **代码走读**：[05-code-walkthrough.md](./05-code-walkthrough.md)
- **API 参考**：[07-api-design.md](./07-api-design.md)
- **开发者指南**：[10-appendix.md §开发者上手指南](./10-appendix.md)
- **架构决策**：[10-appendix.md §ADR](./10-appendix.md)

---

## 项目规模

- 5 个 npm workspace 包
- ~160+ 源码文件
- 43 个 LLM 提供商
- 18 个 API 协议实现
- 319 个测试文件
- 29 篇已有 docs 文档

---

☕️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../res/promotion.jpg)