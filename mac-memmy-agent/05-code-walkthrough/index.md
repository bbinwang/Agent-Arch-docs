# 05 · 核心代码走读（Code Walkthrough）

本目录对六大子系统做逐文件、逐函数/逐类的深度走读。建议结合 [04 模块结构](../04-module-structure.md) 与 [03 流程图](../03-flows-and-sequences.md) 阅读。

| 子系统 | 文件 | 覆盖要点 |
| --- | --- | --- |
| Memory 服务 | [01-memory-service.md](./01-memory-service.md) | HTTP 路由、`MemoryService` 编排、混合检索 `RetrievalService`、Worker/演化、向量存储、Schema |
| Agent Runtime | [02-agent-runtime.md](./02-agent-runtime.md) | `AgentLoop` 状态机、`AgentRunner` 迭代、工具体系、MCP、Skills、三级压缩、记忆 Hook |
| 本地后端 | [03-local-backend.md](./03-local-backend.md) | Fastify 启动/CORS/鉴权、13 路由组、六边形分层、服务编排 |
| Agent 来源 | [04-agent-sources.md](./04-agent-sources.md) | 7 个来源适配器契约与实现、AgentAdapter 插件系统、Skill 写入 |
| 桌面前端 | [05-frontend.md](./05-frontend.md) | React 状态切片、运行时配置发现、主要页面、SSE |
| Electron 外壳 | [06-electron-shell.md](./06-electron-shell.md) | `boot()` 编排、IPC、窗口模式、打包配置、托管 Chromium |

---

☕️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../../res/promotion.jpg)