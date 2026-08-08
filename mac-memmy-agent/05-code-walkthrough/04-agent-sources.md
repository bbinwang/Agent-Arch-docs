# 05.4 · Agent 来源与适配器走读

`App/backend/src/adapters/outbound/` 下有两套相关但独立的适配器系统：**agent-source**（被动扫描外部 Agent 本地历史的读取适配器）与 **agent-adapter**（更高层的 manifest+动态导入插件注册表）。另有 **skill-writer**（为外部 Agent 安装 Hook/Skill/插件）。

## 5.4.1 agent-source 统一契约

`agent-source/types.ts` 定义 `SourceAdapter`：
- `descriptor: SourceDescriptor`(`sourceId`/`displayName`/`builtin`/`dataPath`)
- `detect(): Promise<boolean>`
- `async *scan(options): AsyncIterable<ConversationMessage>`
- `ConversationMessage{messageId,sourceId,conversationId,role,content,createdAt,workspacePath,gitRoot,rawMeta}`
- `ScanOptions{since,maxMessages,maxScanTargets,order,signal,onProgress}`

注册表 `src/services/builtin-agent-source-registry.ts` `createBuiltinAgentSourceRegistry()` 按序实例化 7 个：cursor、claude_code、codex、opencode、openclaw、hermes、workbuddy。平台路径在 `agent-paths.ts`（尊重 `CLAUDE_CONFIG_DIR`/`CODEX_HOME`/`OPENCODE_CONFIG_DIR`/`OPENCLAW_STATE_DIR`/XDG 等）。

### 共享助手（`agent-source/`）
- `jsonl-lines.ts` `readJsonlObjects(filePath,signal)`：流式产出合法 JSON 对象行，跳过坏行/非对象行。
- `conversation-window.ts` `collectConversationWindow(input,since,signal,maxMessages)`：先整缓冲一个扫描目标，再保留**最后消息 `createdAt ≥ since`** 的整会话（保证水位不截断开头用户消息），按会话粒度遵守 `maxMessages`；`remainingMessageCapacity`。
- `secret-redactor.ts` `redactSecrets(content)`：对每条产出消息脱敏。
- `source-registry.ts`：内存 `SourceRegistry`（`list/get/require`）。
- `onboarding-insight-samplers.ts`+`insight-sampler-types.ts`：各 Agent 的洞察采样器。

所有适配器遵循同一骨架：`detect()` 查路径存在 → `scan()` 发现目标 → 每目标用 `collectConversationWindow(...)` 包原始读取 → 每条消息过 `redactSecrets` 映射成 `ConversationMessage`，遵守 `maxMessages`/中止并发进度阶段（`discover→read→redact→emit→done`）。

### 7 个适配器（`agent-source/<name>/`）

| 适配器 | 来源位置 | 读取器/格式 | 要点 |
| --- | --- | --- | --- |
| **claude-code** | `~/.claude/projects/`（按项目会话目录） | `transcript-reader.ts`(JSONL via `readJsonlObjects`)；`project-discovery.ts` 找会话文件 | 解析 `type:"user"\|"assistant"`，提取 `message.content`（字符串或 `{type:"text",text}` 数组），用 `record.uuid`/`sessionId`/`cwd` |
| **codex** | `~/.codex/sessions/` | `rollout-reader.ts`(JSONL) | 每行 `type:"response_item"` 带 `payload`；`payload.type==="message"`→user/assistant/developer(→system)，`function_call`/`function_call_output`/`web_search_call`→合成**工具**消息；`session-discovery.ts` 列文件，会话 id 取自文件名 UUID |
| **cursor** | VSCode/Cursor `workspaceStorage/<hash>/state.vscdb` + 全局 `globalStorage/state.vscdb` | `vscdb-reader.ts`(SQLite `node:sqlite` 只读) | 读两表：`ItemTable`(key→JSON 消息数组/容器)与 `cursorDiskKV`(键 `bubbleId:<convId>:<bubbleId>`，bubble `type` 1=user/2=assistant)；`workspace-discovery.ts` 把存储 hash 映射到 workspace/git root |
| **openclaw** | `~/.openclaw/` state dir | `db-reader.ts`(SQLite) | 两 schema：会话(`messages`+`conversations`，role user/assistant)与记忆(`chunks` 表，动态探列含 `dedup_status`)；`db-discovery.ts` 枚举 DB(`schemaKind: conversation\|memory`) |
| **opencode** | `~/.local/share/opencode/opencode.db` | `db-reader.ts`(SQLite join `message ⋈ session ⋈ part`) | 从 `part.data` 文本部分(`type:"text"`)组装消息文本；workspace/git-root 取自 `message.data.path.{cwd,root}` 或 `.git` 回溯 |
| **hermes** | `~/.hermes/`(rollout JSONL + `state.db`) | `rollout-reader.ts`(JSONL，容忍多种记录形态)+ `state-db-reader.ts`(SQLite `messages`，可选列 `tool_call_id/tool_calls/tool_name/platform_message_id`，左连 `sessions` 取 `cwd`) | 合并两种目标 |
| **workbuddy** | `~/<workbuddy home>/projects/` | `history-reader.ts`(自有 JSONL 流，非共享助手) | 富事件模型 `message`/`function_call`/`tool_call`/`function_call_result`/`tool_result`；剥离 WorkBuddy 系统 XML 标签（`<additional_data>`/`<system_reminder>`/`<user_query>`），过滤内部/compact 消息 |

## 5.4.2 agent-adapter 插件系统

这是与 agent-source **不同**的高层适配器系统：基于 manifest + 动态 import 的插件注册表，服务于 `AgentAdapter` 契约（`detect/validateSource/scan/installSkill/removeSkill`）。

### 类型（`types/`）
- `domain.ts` — `BuiltinAgentKind = "cursor"|"codex"|"claude_code"|"opencode"|"openclaw"|"hermes"|"workbuddy"`；`AgentKind` 对第三方开放；`AgentMessage{role,content,createdAt?,metadata?}`、`AgentScanRecord{sourceExternalId,sourceHash,agentKind,workspacePath?,gitRoot?,messages,nextCursor?}`。
- `adapter.ts` — `AgentAdapter` 接口：`kind`、`descriptor`、`detect(input)→DetectedAgentSource[]`、`validateSource`、`scan(input)→AsyncIterable<AgentScanRecord>`、`installSkill`、`removeSkill`；`AgentAdapterCapabilities{detect,scan,installSkill,removeSkill}`(布尔)。
- `plugin.ts` — `AgentAdapterPluginManifest{id,kind,displayName,version,modulePath,enabled,priority,capabilities}`；`AgentAdapterPluginFactory=(ctx:{manifest})=>AgentAdapter`；`AgentAdapterPluginSource.loadManifests()` 与 `AgentAdapterModuleLoader.importModule(path)` 是两个可插拔缝。
- `json.ts` — JSON 基元/对象/值助手。

### 装载/注册机制
- `plugin-source.ts` `createDirectoryAgentAdapterPluginSource({pluginDirectories,fileSystem?})`：每个插件目录扫子目录（期望含 **`agent-adapter.plugin.json`**——清单文件名常量 `AGENT_ADAPTER_PLUGIN_MANIFEST`）或 `*.agent-adapter.json`；解析 manifest，`modulePath` 相对清单目录；按 `priority` 降序、`id` 升序排。
- `manifest.ts` `parseAgentAdapterPluginManifest(value)` 严格解析；`createAgentAdapterDescriptor(manifest)` 派生轻量 descriptor；`isBuiltinAgentKind` 仅分类不拒绝第三方。
- `plugin-loader.ts` `createAgentAdapterPluginLoader()` 用 `createDynamicImportModuleLoader()`(`import(modulePath)`)：对每个 manifest 动态导入、解析工厂（`createAdapter` 命名导出→`default`→`default.createAdapter`）、调 `createAdapter({manifest})`，校验 `adapter.kind===manifest.kind` 与 `descriptor.id===manifest.id`。
- `registry.ts` `createAgentAdapterRegistry({pluginSource,pluginLoader})` 惰性加载并缓存：`list()`、`get(kind)`、`detectAll(input)`（仅对 `capabilities.detect` 为真的跑 detect）、`reload()`；`loadAdapters` 过滤 `enabled` 并断言 `id`/`kind` 唯一；`createDefaultAgentAdapterRegistry({pluginDirectories})` 默认插件目录为 `<本文件目录>/plugins`。

在 `src/index.ts` 装配为 `BackendServices.agentAdapterRegistry`。（源码中 `plugins/` 目录默认不存在，构建时若有则创建。）

## 5.4.3 skill-writer（实时接入写入）

`skill-writer/` 把 Memmy skill 文件/插件写入各 Agent 配置。`types.ts` 定义 `SkillTarget{targetId,resolveRootDirectory(),install/uninstall/isInstalled,installPlugin?/uninstallPlugin?/detectMemoryPluginConflict?}` 与 `SkillManifest{targetId,content,marker}`；`target-registry.ts createSkillTargetRegistry(targets)`。各 Agent 具体目标在 `claude-code/codex/cursor/hermes/openclaw/opencode/workbuddy/` 子目录（加 `templates/`、`hook-command.ts`、`memmy-runtime-config.ts`、`skill-directory.ts`）。在 `services/index.ts` 组装 7 个 target，由 `SkillDistributionService` 消费，对应路由 `agent-sources.ts` 的 `:sourceId/skill` 与 `:sourceId/plugin`。

### 各 Agent 实时接入写入（摘要）
- **Cursor**：写 `~/.cursor/hooks.json`（追加，不覆盖）、`~/.cursor/hooks/memmy-resume-hook.mjs`、`memmy-memory-config.json`、`~/.cursor/skills/memmy-memory/SKILL.md`。
- **Claude Code**：`~/.claude/settings.json`、`hooks/memmy-resume-hook.mjs`、`memmy-memory-config.json`、`commands/memmy-resume.md`、`CLAUDE.md`、`skills/memmy-memory/SKILL.md`。
- **Codex**：`~/.codex/hooks.json`、`hooks/memmy-resume-hook.mjs`、`memmy-memory-config.json`、`AGENTS.md`、`skills/memmy-memory/SKILL.md`。
- **OpenCode**：`~/.config/opencode` 下 `plugins/memmy-memory.js`、`memmy-memory-config.json`、`commands/memmy-resume.md`、`AGENTS.md`、`skills/memmy-memory/SKILL.md`（尊重 `OPENCODE_CONFIG_DIR`/`XDG_CONFIG_HOME`）。
- **OpenClaw**：`~/.openclaw/extensions/memmy-memory/`、`openclaw.json`（`plugins.slots.memory` 指向 memmy-memory）、`skills/memmy-memory/SKILL.md`、workspace `AGENTS.md`；memory slot 唯一，冲突先确认。
- **Hermes**：`~/.hermes/plugins/memmy-memory/`、`memmy-resume/`、`config.yaml`（`memory.provider=memmy-memory`）、`SOUL.md`、`skills/memmy-memory/SKILL.md`；Memory Provider 唯一。

> 设计评价：**优点**：统一 `SourceAdapter` 契约让"加一种 Agent 来源"成本可控；`collectConversationWindow` 的整会话+水位≥设计是正确性关键；脱敏前置；高层 `agent-adapter` 插件化为第三方扩展留了缝；实时接入文件路径与官方文档完全一致。**潜在改进**：两套适配器系统（agent-source vs agent-adapter）职责重叠易混淆，值得在文档/命名上更强地区分；workbuddy 用自有 JSONL 流而非共享 `readJsonlObjects`，存在重复实现。

---

> ← [走读索引](./index.md) ｜ 上一个 → [03 本地后端](./03-local-backend.md) ｜ 下一个 → [05 桌面前端](./05-frontend.md)

---

☕️ 制作不易，请我喝咖啡☕️关注我➕