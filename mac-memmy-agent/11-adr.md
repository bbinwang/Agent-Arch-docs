# 11 · 架构决策记录（ADR）

本章记录 memmy-agent 的关键架构决策及其背景、理由与后果。编号便于引用；状态均为当前代码所体现的既定决策。

---

## ADR-0001 · 采用本地优先（Local-First）架构

- **状态**：已采纳
- **背景**：产品核心卖点是"数据主权"——个人记忆、偏好、项目经验必须留在用户机器，且要能离线可用。
- **决策**：记忆/配置/App 状态默认全部落 `~/.memmy`（SQLite），扫描与入库本地完成；云只承担账号/配额/ASR/法务/遥测与账号模式模型网关。
- **理由**：契合目标用户（重度开发者、重视隐私者）；降低云依赖与合规风险。
- **后果**：需要本地 SQLite + 向量能力（sqlite-vec）；安全边界以"本机运行 + 显式认证"为基础；跨设备同步需未来额外设计。

## ADR-0002 · Memory 服务用原生 `node:http` 而非 Web 框架

- **状态**：已采纳
- **背景**：Memory 服务需轻量、可独立二进制分发、依赖极少。
- **决策**：用 `node:http` + 手写 `routeRequest` + `API_ROUTES` 声明式路由表。
- **理由**：零框架依赖、启动快、便于打包成独立二进制（`build-binary.sh`）；路由表集中可审计。
- **后果**：需自实现鉴权/CORS/SSE/错误信封；换来可移植性与最小依赖面。

## ADR-0003 · 后端用 Fastify + `node:sqlite`，Memory 用 `better-sqlite3`

- **状态**：已采纳
- **背景**：后端需高吞吐 HTTP + 多账户 App 状态；Memory 需向量扩展与成熟同步驱动。
- **决策**：`@memmy/backend` 用 `fastify` + Node 内置 `node:sqlite`(`DatabaseSync`)；`@memmy/memory` 用 `better-sqlite3` + `sqlite-vec`。
- **理由**：后端 App 状态无需向量扩展，用内置驱动减少原生依赖；Memory 需 `sqlite-vec` 与 `better-sqlite3` 的成熟生态。两者物理隔离（不同库、不同进程）。
- **后果**：存在两套 SQLite 访问方式，增加心智负担（有意隔离，见 ADR-0011）。

## ADR-0004 · AgentLoop 与 AgentRunner 拆分 + Hook 扩展脊梁

- **状态**：已采纳
- **背景**：一轮任务涉及回合生命周期、会话、调度、MCP、工具、模型调用、记忆，复杂度高。
- **决策**：外层 `AgentLoop`（回合状态机 + 会话 + 调度 + 检查点）+ 内层 `AgentRunner`（迭代循环：预处理→调模型→执行工具）；所有横切扩展收敛到 `AgentHook`，由 `CompositeAgentHook` 组合。
- **理由**：事件驱动（网关）与同步（CLI/SDK/API）路径复用同一 `processMessageInternal`；记忆/BYOK/进度/子 Agent 都是 Hook，扩展不改核心。
- **后果**：`loop.ts` 较大；状态机与调度耦合较深（见技术债）。

## ADR-0005 · 记忆用四层模型 + 混合检索 + 后台演化

- **状态**：已采纳
- **背景**：要让记忆"越用越懂你"且可检索、可复用，又要避免噪声与幻觉。
- **决策**：L1 Trace / L2 Policy / L3 World Model / Skill 四层；检索 6 通道（向量×3/FTS/pattern/structural）+ RRF + MMR + LLM 过滤；写入即入队，摘要/向量化/演化全在后台 Worker 异步推进。
- **理由**：分层匹配不同召回场景；混合检索兼顾语义与词法；后台化避免阻塞 Agent 回合。
- **后果**：算法参数众多、调参门槛高；演化依赖模型质量。

## ADR-0006 · 跨 Agent 共享记忆 = 统一来源契约 + 三条写入路径

- **状态**：已采纳
- **背景**：需接管多种外部 Agent 的历史并实时接入其新对话。
- **决策**：统一 `SourceAdapter`(`detect`+`async *scan`) 契约产出 `ConversationMessage`；扫描/回合采集/手动 `add` 三路径最终写同一 `memory.sqlite`；实时接入靠为各 Agent 安装原生 Hook/插件 + 精简 Skill。
- **理由**：加一种 Agent 来源成本可控；统一存储天然实现共享。
- **后果**：需处理各 Agent 异构格式（JSONL/SQLite）与配置侵入；整会话水位扫描保证正确性。

## ADR-0007 · 桌面本地 API 用临时端口 + runtime.json

- **状态**：已采纳
- **背景**：桌面后端与前端/CLI/Hook 需安全通信，但固定端口易冲突且多实例难共存。
- **决策**：后端绑 `127.0.0.1:0`（临时端口），把 `{baseUrl,localToken}` 写 `~/.memmy/runtime.json`（`chmod 0600`，目录 `0700`）；前端经预加载桥 `window.memmy.getRuntimeConfig()` 读取（Vite env/`__memmy_runtime_config` 兜底）。
- **理由**：避免端口冲突、支持多实例、token 随机化保障本机安全。
- **后果**：固定端口仅 Vite `19000`；CLI/Hook 依赖 `runtime.json` 发现服务。

## ADR-0008 · Account 与 BYOK 双模式 + 热切换

- **状态**：已采纳
- **背景**：既要"零配置试用"（账号 Token），又要"用自己的 Key"（BYOK）。
- **决策**：双模式；切换重写 `~/.memmy/config.yaml`；`AgentLoop` 用重载快照加载器，`refreshProviderSnapshot()` 即时拾取新 provider/model/preset，无需重启。
- **理由**：降低试用门槛同时满足重度用户；热切换提升体验。
- **后果**：配置投影逻辑较复杂；需 `mapModelProtocol` 映射协议名。

## ADR-0009 · 工具/能力插件化（Tools + MCP + Skills + AgentAdapter）

- **状态**：已采纳
- **背景**：Agent 能力需持续扩展，且要接入第三方工具与外部 Agent。
- **决策**：内置工具可经 `memmyAgent.tools` 字段被发现；MCP 支持 stdio/sse/streamableHttp/inMemory；Skills 目录化（workspace 覆盖 builtin）；AgentAdapter 用 `agent-adapter.plugin.json` 清单 + 动态 import 注册。
- **理由**：开放的扩展生态；浏览器工具复用 `@playwright/mcp` 进程内 MCP。
- **后果**：多套扩展机制并存，学习曲线略陡。

## ADR-0010 · 记忆 fail-open + 显式错误（无假记忆）

- **状态**：已采纳
- **背景**：记忆服务可能短暂不可用，但不能因此中断 Agent 正常对话，更不能编造"假记忆"。
- **决策**：召回/采集失败仅记录错误并继续（fail-open）；Memory 不可用时显式报错；Memory client 必须连真实服务或发现本地 SQLite，否则启动失败。
- **理由**：可用性与可信度并重。
- **后果**：用户需经日志/`--verbose` 察觉失效（已在可观测性上补强）。

## ADR-0011 · `App/memmy-agent` 独立 npm 项目（独立 lock）

- **状态**：已采纳
- **背景**：Agent Runtime 依赖庞大（多渠道 SDK、Playwright、多 AI SDK），放入根 workspaces 会拖慢安装与 CI。
- **决策**：`App/memmy-agent` 不在根 workspaces，有独立 `package-lock.json`，由 `dev-start.sh` 用 `npm ci --prefix` 驱动。
- **理由**：隔离重型依赖、加速根安装、便于 Agent Runtime 独立演进。
- **后果**：版本同步需 `sync-project-version.mjs` 显式处理；构建编排略复杂。

## ADR-0012 · 发布制品走 OSS 预构建 + GitHub Release 校验

- **状态**：已采纳
- **背景**：跨平台原生模块在 GitHub Actions 直接打包成本高、慢。
- **决策**：本地/自有流水线产出签名安装包上传 OSS；`github-release.yml` 下载并用 `Content-MD5` 校验后发布 GitHub Release（draft→latest）。
- **理由**：复用预构建、保证完整性、发布说明可前置 release-notes。
- **后果**：发布依赖 OSS 可用性与签名流水线。

---

> 上一节 ← [10 开发者上手指南](./10-developer-guide.md) ｜ 下一节 → [12 关键算法与测试策略](./12-algorithms-testing.md)

---

# ❤️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../res/promotion.jpg)