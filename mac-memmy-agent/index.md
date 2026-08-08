# Memmy-Agent 架构设计文档

> 仓库：[`MemTensor/memmy-agent`](https://github.com/MemTensor/memmy-agent) ｜ 版本：`1.0.4` ｜ 生成日期：2026-07-31
> 口号：**Memmy is your personal memory hub — and a dedicated Agent that knows you best.**
> 一句话定位：**本地优先（local-first）的跨 Agent 个人记忆底座 + 通用 Agent 运行时**。

本文档是对 memmy-agent 整个仓库的全面、深入、基于真实源码的架构解读。全部内容严格基于仓库 `App/`、`Memory/`、`Migrations/`、`scripts/`、`tests/` 下的实际代码、配置与官方文档分析得出，所有图表均使用标准 Mermaid 语法，可直接渲染。

---

## 文档地图

| 序号 | 文件 | 内容 | 适合谁读 |
| --- | --- | --- | --- |
| 01 | [项目概述](./01-overview.md) | 定位、技术栈、架构风格、功能与非功能性需求 | 所有人（先读） |
| 02 | [C4 架构模型](./02-c4-architecture.md) | Context / Container / Component / Code 四层视图 | 架构师、技术负责人 |
| 03 | [系统流程与时序图](./03-flows-and-sequences.md) | 10 个核心业务流程与关键时序 | 后端/运行时开发 |
| 04 | [模块结构与依赖](./04-module-structure.md) | 目录树、模块职责、依赖关系图 | 新加入的开发 |
| 05 | [核心代码走读](./05-code-walkthrough/) | 六大子系统逐文件/逐函数讲解 | 深度二次开发 |
| 06 | [数据模型与数据库设计](./06-data-model.md) | SQLite 表结构、ER 图、索引、事务 | 数据/存储相关 |
| 07 | [API 与接口设计](./07-api-design.md) | Memory API / 本地 API / OpenAI 兼容 API / Cloud API | 集成/对接开发 |
| 08 | [部署、运维与基础设施](./08-deployment-ops.md) | 启动编排、打包、CI/CD、监控日志 | 运维、发布 |
| 09 | [改进建议、风险与未来规划](./09-risks-roadmap.md) | 优缺点、技术债、风险、Roadmap | 架构评审 |
| 10 | [开发者上手指南](./10-developer-guide.md) | 本地运行、调试、测试流程 | 新贡献者 |
| 11 | [架构决策记录（ADR）](./11-adr.md) | 关键决策的历史与理由 | 架构考古 |
| 12 | [关键算法与测试策略](./12-algorithms-testing.md) | 混合检索/RRF/MMR/奖励传播、复杂度、测试 | 算法/质量工程 |

## 推荐阅读路线

- **想 10 分钟看懂全局**：01 → 02（C4 的 Context + Container）→ 04（模块图）。
- **要做二次开发**：01 → 02 → 04 → 对应的 `05-code-walkthrough/` 子文档 → 10（上手指南）。
- **要对接 Memory 能力**：07（API）→ 06（数据模型）→ 12（检索算法）。
- **要做发布/运维**：08 → 11（ADR）→ 09。

## 文档约定

- 路径一律相对仓库根（如 `App/backend/src/index.ts`）。
- 端口均为默认值：Memory `18960`、Gateway 健康 `18970`、WebUI/WebSocket/Admin `18980`、OpenAI 兼容 API `18990`、桌面 Vite `19000`、HMR `19010`、桌面本地 API为临时端口。
- 所有 Mermaid 图后均附详细中文文字解释。

---

☕️ 制作不易，请我喝咖啡☕️关注我➕