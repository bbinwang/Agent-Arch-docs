# 05.2 · Agent Runtime 代码走读

包：`memmy-agent`（`App/memmy-agent/）。提供 `memmy` CLI、交互 TUI、OpenAI 兼容 API、渠道网关，核心是 `AgentLoop` + `AgentRunner`。

## 5.2.1 入口与 CLI

- `src/main.ts` — 进程入口（`#!/usr/bin/env node`）。先 `./load-env.js` 注入 `MEMMY_CLOUD_SERVICE`（从仓库根 `.env`），再调 `src/entrypoints/cli/commands.ts` 的 `main(argv)`。
- `src/entrypoints/cli/commands.ts` — 用 **commander**（`new Command("memmy")`）。`main` 先处理三种根级（无子命令）情形：`internal browser-prepare`、`-V/--version`、裸调用→交互 TUI；否则注册子命令：`onboard`（写配置+工作区模板，`--wizard` 走 `onboard.ts`）、`serve`（OpenAI 兼容 API，host/port 来自 `config.api`）、`gateway`（长驻渠道守护，装配 `MessageBus`/`ChannelManager`/`CronService`/`HeartbeatService` + 循环 `run()`）、`agent [-m]`（单轮；TTY 且无 `-m` 时进 REPL `runInteractiveAgent()`）、`status`/`config set`/`channels`/`plugins`/`provider`。
- `src/entrypoints/cli/tui.tsx` — Ink/React TUI；`stream.ts` 提供 `StreamRenderer`/`ThinkingSpinner`。
- `src/memmy-agent.ts` — SDK 入口，导出 `MemmyAgent`（别名 `Memmy`）包装 `AgentLoop`；`fromConfig(...)`、`.run(message,{sessionKey})`→`loop.processDirect`。

## 5.2.2 AgentLoop — 回合编排状态机

`src/core/agent-runtime/loop.ts`，`class AgentLoop`（~2377 行）。`new AgentLoop(init)` 或 `AgentLoop.fromConfig(config,bus,extra)`。持有：`provider`、`model`、`sessions: SessionManager`、`runner: AgentRunner`、`context: ContextBuilder`、`tools`、`consolidator: Consolidator`、`autoCompact: AutoCompact`、`dream: Dream`、`subagents: SubagentManager`、`browserSessionManager`、`cronService`、`sessionDagQueue`，以及 MCP 状态（`mcpServers`/`mcpStacks`/`mcpConnected`）。

**回合状态机**（`TurnState` + `TurnContext`）：
```
RESTORE → COMPACT → COMMAND → BUILD → RUN → SAVE → RESPOND → DONE
```
由 `stateRestore/stateCompact/stateCommand/stateBuild/stateRun/stateSave/stateRespond` 实现，`processMessageInternal()` 串联。

**两条入口路径：**
- `processDirect(content,opts)`（同步，CLI/SDK/API 用）直接调 `processMessageInternal`。
- `run()` → `dispatchMessage()`（事件驱动，网关用）：从 `bus.inbound` 拉 `InboundMessage`，**每会话 `AsyncMutex` + `AsyncQueue`** 排队待决消息、可取消的 dispatch 任务。

`runAgentLoop()` 包装 `runner.run(AgentRunSpec)`，返回 `[finalContent, toolsUsed, allMessages, stopReason, hadInjections, finalContentStreamed]`。

**检查点/恢复**：`setRuntimeCheckpoint`/`restoreRuntimeCheckpoint`（把进行中的助手消息 + 已完成/待决工具调用持久化进 session metadata）与 `markPendingUserTurn`/`restorePendingUserTurn`，用于恢复被中断的回合。

**并发控制**：每会话 `AsyncMutex`（`lockFor`）、`pendingQueues`、`activeTasks`（可取消）、`cancelActiveTasks()`、`waitForCronTargetAvailable()`、`isSessionBusy()`、`isSessionGoalActive()`。

## 5.2.3 AgentRunner — 迭代循环

`src/core/agent-runtime/runner.ts`，`class AgentRunner.run(spec: AgentRunSpec): Promise<AgentRunResult>`。核心迭代：

```
for (iteration < spec.maxIterations):
  预处理消息:
    dropOrphanToolResults → backfillMissingToolResults
    → microcompact → applyToolResultBudget → snipHistory(按 token 窗口)
    → 再 drop/backfill
  requestModel(): 调 provider
  if response.shouldExecuteTools:
    push 助手消息; executeTools()(批量; 并发只读, 串行独占)
    追加工具结果消息; checkpoint
    drain injectionCallback(待决用户队列 + goal-continue); continue
  else:
    终结(空→requestFinalizationRetry; length→长度恢复; 错误/空终态); drain; break
```

- **Provider 调用 `requestModel`**：支持流式（`wantsStreaming`）、进度增量流式、非流式带 `withTimeout`/`withAbort`；`onContentDelta`/`onThinkingDelta`/`onToolCallDelta` 钩子；LLM 墙钟超时取自 `MEMMY_AGENT_LLM_TIMEOUT_S`（默认 300s）或 `runnerWallLlmTimeoutS`。
- **Hook 体系**：`AgentHook`（`hook.ts`）含 `beforeRun/afterRun/beforeIteration/afterIteration/beforeToolCall/afterToolCall/beforeExecuteTools/onStream/onStreamEnd/emitReasoning/onBuildSystemPrompt/onRegisterTools/sessionStart/sessionEnd`。Loop 把 `AgentProgressHook`（接 `onProgress`/`onStream` 到 bus）与 `extraHooks`（记忆 + BYOK）用 `CompositeAgentHook` 组合。

> 设计要点：Loop/Runner 拆分使事件驱动与同步两条路径复用同一状态机；预处理五步保证 prompt 不超窗且工具结果不"悬挂"；Hook 是唯一扩展脊梁。

## 5.2.4 工具体系

- `tools/base.ts` — 抽象 `Tool`：getters `name/description/parameters`、`execute(params,ctx)`、`toSchema()`→`{type:"function",function:{...}}`、`castParams/validateParams`；可选静态 `enabled(ctx)/create(ctx)/scopes/concurrencySafe/readOnly/exclusive`。`toolParameters(schema)` 装饰器。
- `tools/schema.ts` — JSON schema DSL（`StringSchema`/`ObjectSchema`…）。
- `tools/registry.ts` — `ToolRegistry`：`register/get/has/toolNames/getDefinitions`（内置排在 `mcp_*` 前）、`prepareCall`（校验）、`execute`。
- `tools/loader.ts` — `ToolLoader.discover()` = `BUILTIN_TOOL_CLASSES` + `discoverPlugins()`（扫最近 `node_modules` 中 `package.json` 带 `memmyAgent.tools`/`memmy-agent.tools` 字段的包，加载具名模块收集 `Tool` 子类，`pluginDiscoverable` 默认 true）；`load()` 按 `enabled(ctx)` 与 `scopes("core")` 过滤实例化。
- **内置工具**（`tools/`）：
  - `shell.ts` `ExecTool`(`exec`)：spawn 子进程、`ExecSessionManager` 管长驻 shell、`BUILTIN_DENY_PATTERNS`、SSRF `containsInternalUrl`、可选 `sandbox`。
  - `filesystem.ts`：`ReadFile/WriteFile/EditFile/ListDir`（工作区路径强制、屏蔽设备路径）。
  - `search.ts`：`FindFiles/Grep`。
  - `web.ts`：`WebSearch`(DuckDuckGo)/`WebFetch`(Readability+Turndown)，均经 `validateUrlTarget` SSRF 防护。
  - `browser.ts`+`browser-setup.ts`+`browser-preview.ts`：`browser_navigate/snapshot/click/type/take_screenshot…`，实现为**进程内 MCP server**（`@playwright/mcp` + Playwright，经 `connectInMemoryMcpServer`），`BrowserSessionManager` 按 `{sessionKey,channel,chatId}` 隔离上下文。
  - `image-generation.ts`(`generate_image`)、`apply-patch.ts`、`spawn.ts`、`exec-session.ts`(`list_exec_sessions`/`write_stdin`)、`message.ts`(`MessageTool` 发渠道并抑制常规最终回复)、`cron.ts`、`long-task.ts`(`long_task`/`complete_goal`)、`agent-source.ts`。

## 5.2.5 MCP 集成

`tools/mcp.ts`（基于 `@modelcontextprotocol/sdk`）。`connectMcpServers(servers,registry)` 遍历配置 server，自动判传输（`command`→stdio、URL 以 `/sse` 结尾→sse、否则 streamableHttp），`listTools()` 后以 `mcp_<server>_<tool>` 注册 `MCPToolWrapper`，并注册 resource/prompt wrapper；支持 `enabled_tools` 白名单（默认 `*`）；每次调用 `tool_timeout`（默认 30s）。`connectMissingServers` 幂等；`connectInMemoryMcpServer` 供浏览器工具在进程内托管 Playwright-MCP。`normalizeSchemaForOpenAI`/`convertMcpToolContent` 适配 schema 与内容。

## 5.2.6 Skills

`core/agent-runtime/skills.ts` `SkillsLoader`：根目录 `<workspace>/skills`（source `workspace`）与 `BUILTIN_SKILLS_DIR=src/skills`（source `builtin`）。Skill 是含 `SKILL.md`（可选 YAML frontmatter）的目录；frontmatter 字段 `description`、`requires:{bins,env}`（用 `which`/`process.env` 检查）、`metadata.memmy.always`/`memmy.manualOnly`。方法 `listSkills/loadSkill/loadSkillsForContext/buildSkillsSummary/getAlwaysSkills/findExplicitSkillNames`（匹配用户文本中的 `$skill-name`）。`ContextBuilder.skills` 把 always-on skill 与摘要注入系统提示。内置 skill（`src/skills/`）由 `syncWorkspaceTemplates` 同步进 workspace（onboard/serve/gateway/agent 调用），workspace skill 按名覆盖内置。

## 5.2.7 会话与长任务

- `core/session/manager.ts` `SessionManager`（文件后端 `<workspace>/sessions`），`Session`=消息列表+metadata，`FILE_MAX_MESSAGES=2000`。`getHistory({maxMessages,maxTokens,includeTimestamps})`、`enforceFileCap`（溢出归档到 `MemoryStore.rawArchive`）、`lastSummary`、WebUI 绑定（`WEBUI_PROJECT_ID_METADATA_KEY`/`WEBUI_WORKSPACE_CWD_METADATA_KEY`）。
- **SessionKey 策略**：`effectiveSessionKey()`；`unifiedSession` 开启时全部映射 `UNIFIED_SESSION_KEY="unified:default"`，否则 `<channel>:<chatId>`（如 `cli:direct`/`api:<id>`/`websocket:<id>`/`slack:<chat>:<thread>`）。
- **跨入口接续**：`processDirect`/`processMessage`/网关 `dispatchMessage` 都落到同一 `processMessageInternal` 与同一 `SessionManager`，故 `serve` 起的会话可被 `gateway` 接续。
- **长任务**：`long-task.ts` `LongTaskTool`/`CompleteGoalTool` 在 session metadata 维持 goal 状态（`GOAL_STATE_KEY`），goal 激活时 `AgentRunner` 注入 goal-continue 消息而非终止，`loop.isSessionGoalActive()` 阻塞 cron 投递；`SubagentManager`(`subagent.ts`) 用独立 `ToolRegistry`/`AgentHook` 跑后台子 Agent，`drainPendingQueue` 等待（`subagentPendingWaitMs=300s`）。

## 5.2.8 三级压缩

- **迭代内**（Runner）：`microcompact`（裁剪超 `MICROCOMPACT_KEEP_RECENT=10` 的旧 read/exec/grep/web 结果）、`applyToolResultBudget`（按工具名字符上限）、`snipHistory`（超 `contextWindowTokens - maxOutput - CONTEXT_SAFETY_BUFFER` 丢最旧非系统消息）。
- **回合级**（`core/agent-runtime/memory.ts` `Consolidator`）：`maybeConsolidateByTokens` 在 `stateBuild` 前及 `stateSave` 后台任务再排一次；超窗时把旧消息摘要进 `session.metadata.lastSummary` 并裁剪，保留近期后缀；支持 `summaryMode:"dag"`（经 Session-DAG 队列）；发 WebUI 压缩状态事件；`compactIdleSession` 自动归档。
- **空闲归档**（`autocompact.ts` `AutoCompact`）：`checkExpired()` 扫空闲超 `sessionTtlMinutes` 的会话后台归档，`prepareSession()` 把 `lastSummary` 作为"上次会话摘要"前缀给下一轮。

## 5.2.9 记忆 Hook

`src/memmy-memory/`：`MemmyMemoryClient`(`client.ts`)、`discoverMemmyMemoryConnection`(`discovery.ts`，默认 `127.0.0.1:18960`，从 `MEMMY_MEMORY_URL`/`MEMORY_SERVICE_URL`/`runtime.json`/`config.yaml` 解析)、`resolveMemmyMemoryConfig`、`MemmyMemoryHook`(`hook.ts`)、`register.ts`(`installMemmyMemory`)。`AgentLoop` 构造时 `installMemmyMemory(...)` 把 `MemmyMemoryHook` 推入 `extraHooks`，由 `CompositeAgentHook` 组合。

- **召回**：`MemmyMemoryHook.beforeRun()`（Runner 的 `hook.beforeRun` 触发）调 `client.startTurn(turnId,{sessionId,query:lastUserText})`，返回 `injectedContext`+`sourceMemoryIds`，`injectMemoryContext()` 拼接 `<memmy_memory_context>`，`onBuildSystemPrompt` 追加"Memmy Memory Protocol"。
- **采集**：`afterRun()` 调 `client.completeTurn(...)`（助手回答、规范化工具调用/结果、推理摘要）；`sessionStart/sessionEnd` 开关 Memory 会话。
- **工具**：`registerMemmyMemoryTools`(`tools.ts`) 注册 `memmy_memory_search/get/add`。

## 5.2.10 OpenAI 兼容 API（serve）

`src/entrypoints/openai-like-api/server.ts` `createApp(agentLoop,modelName,requestTimeout)`→`{fetch,context}`。路由：`GET /health`、`GET /v1/models`、`POST /v1/chat/completions`→`handleChatCompletions`。后者接受 JSON（仅单 user 消息）或 multipart（`message`/`session_id`/`model`/`files`），拒绝非配置模型，映射 `callAgent`→`loop.processDirect(text,{sessionKey:"api:<id>"|"api:default",channel:"api",chatId:"default",media,onStream,onStreamEnd})`，支持 `stream:true` SSE（`sseChunk`+[DONE]），每会话 `withSessionLock`、每请求 `withTimeout`，空回复重试一次。`serve()` 用 `http.createServer` 包 web Request/Response。

## 5.2.11 Provider / BYOK / Account

- `providers/registry.ts` `PROVIDERS: ProviderSpec[]`：每项含 `name`/`backend`(`openai_compat`/`anthropic`/`azure_openai`/`openai_codex`/`github_copilot`/`bedrock`)、flags(`isGateway`/`isLocal`/`isOauth`/`isDirect`)、`envKey`/`defaultApiBase`/`resolveHeaders`。含 `memmy_account`（云网关，`safeMemmyAccountApiBase()`→`${MEMMY_CLOUD_SERVICE}/api/agentExternal/v1`，加 `X-Agent-Region`）、OAuth 项（`openai_codex`/`github_copilot`）、本地（`ollama`/`lmstudio`）。
- `providers/factory.ts` `makeProviderCore` 按 preset→provider→spec 实例化对应类（`OpenAICompatProvider`/`AnthropicProvider`/`AzureOpenAIProvider`/`BedrockProvider`/`OpenAICodexProvider`/`GitHubCopilotProvider`），回退 `OpenAICompatProvider`；非 OAuth/本地/直连缺 Key 抛 `ValueError`。
- `providers/base.ts` `LLMProvider` 抽象 + `LLMResponse`/`ToolCallRequest`(`toOpenAIToolCall`)；实现 `chat/chatWithRetry/chatStream/chatStreamWithRetry`、`generation` 设置、`getDefaultModel`、`supportsProgressDeltas`。
- `providers/memmy-account.ts` 区域解析（`MEMMY_APP_EDITION`→cn/intl）与按模型族适配 thinking 风格（qwen→`enable_thinking`，deepseek/glm/kimi→`thinking_type`）。
- **Account↔BYOK 切换**：`config/schema.ts` 定义 `MemoryProfileName`/`ImageGenerationProfileName="account"|"byok"`；切换重写 `config.yaml`；Loop 用**重载快照加载器**（`snapshot-loader.ts` `makeReloadingProviderSnapshotLoader`/`makeReloadingToolsSnapshotLoader`/`makePresetSnapshotLoader`），`runAgentLoop`/`processMessageInternal` 顶部 `refreshProviderSnapshot()` 即时拾取，经 `runtimeModelPublisher`→`publishRuntimeModelUpdate` 发布。
- **BYOK 用量**：`integrations/byok-token-usage/` `installByokTokenUsage` 推 hook 每轮记录；`createByokTokenUsageRecorder` 供网关/WebUI 标题/DAG 计费用。

> 设计评价：**优点**：Loop/Runner 拆分清晰、Hook 统一扩展脊梁、三级压缩与 token 预算、跨入口同状态机、浏览器工具用进程内 MCP 复用 SDK、Account/BYOK 热切换。**潜在改进**：`loop.ts` 近 2400 行偏大、状态机与调度耦合较深，可进一步拆分；工具 SSRF/工作区约束散落在各工具内，可抽象统一安全策略层。

---

> ← [走读索引](./index.md) ｜ 上一个 → [01 Memory 服务](./01-memory-service.md) ｜ 下一个 → [03 本地后端](./03-local-backend.md)

---

# ❤️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../../res/promotion.jpg)