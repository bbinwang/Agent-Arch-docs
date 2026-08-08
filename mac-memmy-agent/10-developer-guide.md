# 10 · 开发者上手指南

## 10.1 环境要求

- **Node.js**：根 `>=20`；Agent Runtime（`App/memmy-agent`）`>=22`。
- **npm**。
- 平台：macOS / Windows（Windows 建议在 Git Bash 运行 `dev-start.sh`）。

## 10.2 一键启动（推荐）

```bash
git clone https://github.com/MemTensor/memmy-agent.git && cd memmy-agent
cp .env.example .env         # 云地址预填，开箱即用
bash scripts/dev-start.sh    # 装依赖→构建→启动全栈
```

`dev-start.sh` 一条命令完成：装依赖、构建 Memory 与 memmy-agent、安装 `memmy`/`memmy-memory` CLI、启动全栈（Memory / Agent API / Gateway / 前端 / 桌面后端）。桌面 App 打开后完成账号登录或 BYOK 即可使用。`MEMMY_CLOUD_SERVICE` 默认 `https://memmy-api.memtensor.cn`，故拷贝 `.env` 即连官方云，无需自建后端或 API Key。

> agent-api/gateway 会等模型配置（account/BYOK）就绪后才启动。

## 10.3 常用命令（仓库根）

```bash
npm install
npm run dev:desktop     # 同时启动桌面前端 Vite 与 Electron 外壳
npm run build           # 构建 Memory 与所有 workspaces
npm run lint            # lint
npm run typecheck       # 类型检查（含 memory:lint）
npm run test            # release-workflow + memory + workspace + tui-cursor 测试
```

## 10.4 单独开发各子系统

### Memory 服务（独立）
```bash
npm run memory:serve:dev -- \
  --host 127.0.0.1 --port 18960 \
  --db ~/.memmy/memory-service/memory.sqlite \
  --config ~/.memmy/config.yaml
```
相关：`memory:build`/`memory:dev`/`memory:serve`/`memory:worker:run`/`memory:test`/`memory:lint`。

### memmy-agent（从源码）
```bash
cd App/memmy-agent
npm install
npm run build
node dist/main.js --help
node dist/main.js serve          # :18990
node dist/main.js gateway
node dist/main.js agent --message "Summarize the current workspace"
npm run typecheck && npm test
```

### 配置初始化
```bash
memmy onboard            # 写 ~/.memmy/config.yaml + workspace
memmy onboard --wizard   # 交互式配置模型/provider/tools/API
memmy status             # 查看配置/workspace/model/provider
```
默认位置：配置 `~/.memmy/config.yaml`、workspace `~/.memmy/workspace`；可用 `MEMMY_CONFIG`/`MEMMY_AGENT_WORKSPACE` 或 `--config`/`--workspace` 覆盖。

## 10.5 最小 BYOK 配置（`~/.memmy/config.yaml`）

```yaml
agents:
  defaults:
    model: openai/gpt-4.1
    provider: openai
    timezone: Asia/Shanghai
providers:
  openai:
    apiKey: ${OPENAI_API_KEY}   # 支持 ${ENV_NAME} / ${ENV_NAME:fallback}
tools:
  browser:
    enabled: true
    maxSessions: 4
    idleTimeoutS: 900
```
```bash
export OPENAI_API_KEY="your-api-key"
```
OAuth provider：`memmy provider login openai_codex` / `memmy provider logout openai_codex`。

## 10.6 CLI 速查

| 任务 | 命令 |
| --- | --- |
| 帮助 | `memmy --help` |
| 初始化 | `memmy onboard` / `memmy onboard --wizard` |
| 状态 | `memmy status` |
| 交互聊天 | `memmy` |
| 单条消息 | `memmy agent --message "..."` |
| OpenAI 兼容 API | `memmy serve`（默认 `:18990`，`/health`、`/v1/models`、`/v1/chat/completions`） |
| 渠道网关 | `memmy gateway`（健康 `:18970`） |
| 渠道/插件 | `memmy channels status`、`memmy channels login <ch>`、`memmy plugins list` |
| 记忆（外部 Agent/脚本） | `memmy-memory init/health/search "q"/add "x"/get <id>`（默认连 `:18960`） |
| 热加载配置 | `memmy-memory reload-config`（改 `storage` 需重启） |

## 10.7 调试技巧

- **检索诊断**：`memmy-memory search "你的查询" --verbose`，关注 `candidateMemoryIds`/`hits`/`sourceMemoryIds`/`status`；记忆管理"日志页"看 `memory.search` 的 candidates/filtered/droppedByLlm 与 `memory.add` 写入字段。
- **日志**：Electron IPC `open-logs-directory`/`export-diagnostics-report`/`get|set-log-level`；Memory `api_logs` 表。
- **本地数据**：默认 `~/.memmy/memory-service`；可经 UI 打开目录、导出 `memory.sqlite`、二次确认后清空。
- **失败行为**：记忆召回/采集失败 fail-open（记录错误继续）；Memory 不可用时显式报错而非假数据。

## 10.8 测试流程

- **单元**：各 workspace `npm test`（vitest）；根 `npm run test` 含 release-workflow + memory + workspace + tui-cursor。
- **类型**：`npm run typecheck`（先 `version:sync`）。
- **smoke（端到端）**：
  - `npm run smoke:memory-layer`：构建 backend → typecheck → `tsx tests/smoke/memory-layer-smoke.ts`，驱动**真实 Memory 服务**（`MemoryDb`+`MemoryService`+stub Embedder），验证 `health`→`openSession`→`completeTurn`(断言 episodeId/rawTurnId/l1MemoryId 且作业含 embedding)→`runWorkerOnce`(failed===0)→`getMemory`(kind=trace, layer=L1)。
  - `npm run smoke:local-agent-memory`（或 `:cursor`）：构建 memory+backend，用 captured-fetch `MemmyMemoryClient` + `registerMemmyMemoryTools` 进 `ToolRegistry`，执行 `memmy_memory_search/get`，验证 Agent↔Memory 工具契约（无真实网络）。
  - 对应计划测试：`tests/smoke/*-smoke-plan.test.ts`（`smoke:memory-layer:test`/`smoke:local-agent-memory:test`）。

## 10.9 打包

```bash
npm run package:mac           # macOS DMG
npm run package:mac:unsigned  # 跳过签名
npm run package:win:x64       # Windows x64
# 以及 cn/intl × signed/unsigned 变体
```

## 10.10 推荐入门路径

1. 读 [01 概述](./01-overview.md) + [02 C4](./02-c4-architecture.md) 建立全局观。
2. `bash scripts/dev-start.sh` 跑起来，在桌面端发一条任务、扫描一个 Agent 来源。
3. 读 [05.2 Agent Runtime](./05-code-walkthrough/02-agent-runtime.md) 理解一轮推理；读 [05.1 Memory](./05-code-walkthrough/01-memory-service.md) 理解召回。
4. 改一个小工具或调一个检索参数，`npm run smoke:memory-layer` 验证。

---

> 上一节 ← [09 改进建议与风险](./09-risks-roadmap.md) ｜ 下一节 → [11 架构决策记录（ADR）](./11-adr.md)

---

☕️ 制作不易，请我喝咖啡☕️关注我➕