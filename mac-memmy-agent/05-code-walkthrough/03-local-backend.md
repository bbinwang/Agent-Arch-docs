# 05.3 · 本地后端代码走读

包：`@memmy/backend`（`App/backend/）。Fastify + SQLite（`node:sqlite`），六边形架构。本章按"启动 → 入站 API → 出站适配器 → 基础设施 → 服务层 → 权限/埋点"走读。

## 5.3.1 启动与入口

- `src/index.ts` — `createLocalBackend(options)` 构建并返回后端实例；`createLocalApiServer` 建 Fastify 但**不绑端口**，由启动器 `server.listen({host:"127.0.0.1",port:0})` 绑临时端口，写入 `~/.memmy/runtime.json`(`baseUrl`+`localToken`)。还装配 `createDefaultAgentAdapterRegistry({pluginDirectories})`、`createDefaultMemoryClient`、`createDefaultCloudClient`（含 `deviceId`）。
- `src/load-env.ts` — `loadCloudServiceEnv` 加载 `.env`。
- `src/config/service-urls.ts` — `resolveCloudClientConfig(env)`→`{baseUrl: env.MEMMY_CLOUD_URL||resolveCloudServiceBaseUrl(env.MEMMY_CLOUD_SERVICE), timeoutMs: 5000}`。

## 5.3.2 入站本地 API（Fastify）

### `src/adapters/inbound/local-api/server.ts`
`createLocalApiServer(options)` 建 Fastify（`logger:false`）。
- **CORS**（`onRequest`，`applyCorsHeaders`/`isAllowedOrigin`）：允许 `file://`、opaque `"null"`、host ∈ `{127.0.0.1,localhost,::1,[::1]}`；`OPTIONS`→204；非法 Origin（带 `Origin`）→403 `forbidden_origin`；`allowedOrigins` 可覆盖。
- **鉴权**：受保护路由 `preHandler`=`authenticateRuntimeToken`，读 header `x-memmy-local-token` 调 `permissionManager.verifyRuntimeToken`（`timingSafeEqual`，token 为随机 UUID）。SSE 因 EventSource 不能设头，允许 `?token=` 传同一 token。Composio MCP 桥用**独立** token（`mmt_<random>`，header `x-memmy-mcp-token`）。
- 独立处理器：`GET /api/health`、`GET /api/app/bootstrap`（受保护）、`GET /api/events`（hijacked 长连 SSE，15s 心跳，转发 `progressBus` 的扫描进度/完成事件）。

### 路由组（`routes/`）
每个 `register*Routes(app,{…})`，请求/响应用 `@memmy/local-api-contracts` 的 zod schema 校验，错误经 `withErrorEnvelope`。

| 文件 | 关键端点 |
| --- | --- |
| `account.ts` | `send-code`/`verify-code`/`invitation`/`profile`/`guide-finished`/`logout`/`session`/`avatars` |
| `agent-runtime/` | 聚合 7 子模块，前缀 `/api/v1/`：`admin`(`reload-config`)、`health`、`sessions`(open/close)、`turns`(start/complete)、`search`、`memory`(add/logs/processing/get/delete)、`panel`(overview/analysis/items/tasks) |
| `agent-sources.ts` | 发现+扫描管线：`GET /api/agent-sources`、memory-plugin-conflicts、auto-inject/run、scan/status、`POST .../scan`(权限 `canScanAgentSource`)、scan/stop、scan/cancel、manual、`:sourceId/managed/{import,sync}`、PATCH/DELETE managed、`POST/DELETE :sourceId/skill`、`POST/DELETE :sourceId/plugin`。扫描可**内联**或跑 **worker_thread**（`agent-source-scan-worker`），用 scan journal 持久化续跑 |
| `app-config.ts` | `PATCH /api/app/{settings,privacy,scan-preferences,onboarding,improvement-program,skin}`、token-usage、model-config{,/test} |
| `asr.ts` | `POST /api/asr/transcriptions`（云 ASR 代理） |
| `byok-token-usage.ts` | `POST /events`、`GET /summary` |
| `channels.ts` | `/api/v1/channels/...`（定义/连接/`:provider/connect` 长轮询/disconnect），经 `MemmyAgentAdminClient` |
| `composio-mcp.ts` | MCP 桥 `/mcp/composio`(POST/GET/DELETE)，`x-memmy-mcp-token` 鉴权；建含 3 元工具（`composio_search_tools`/`get_tool_schemas`/`execute_tool`）的 MCP Server，每请求一个无状态 `StreamableHTTPServerTransport` |
| `integrations.ts` | Composio：capabilities、`:slug/authorize`、connections、`connections/:id` |
| `local-data.ts` | 本地 SQLite 导出/打开目录/清空 |
| `onboarding-insight.ts` | `POST .../insight-report` 与 `.../stream`(SSE)；`scan_only`/`scan_and_write_skill` 才生成，否则 `skipped` |
| `token-quota.ts` | `GET /eligibility`、`POST /request` |

## 5.3.3 出站适配器

- **`agent-source/`** — 7 个来源适配器 + 共享助手（见 [05.4](./04-agent-sources.md)）。
- **`agent-adapter/`** — AgentAdapter 插件系统（见 [05.4](./04-agent-sources.md)）。
- **`cloud-client/`** — `http-cloud-client.ts`(`createHttpCloudClient`) POST 到 `${baseUrl}/api/agentUser/...`、`/api/composio/...`、`/api/agentAsr/transcriptions`；解析云信封 `{code,message,data}`（`0`=成功），业务码映射本地错误码（`unauthorized 40100`/`forbidden 40101`/限流 `40300-40400`）；带 `lang`、`authorization: Bearer`、`x-memmy-composio-token`、`x-memmy-device-id`、`x-agent-region`；Composio 路由 60s 超时。
- **`memmy-agent-admin-client/`** — `HttpMemmyAgentAdminClient` 访问本地 Agent WebUI `18980`：`GET /webui/bootstrap`（带 `x-memmy-agent-auth:<bootstrapSecret>`，来自 `config.yaml` 的 `channels.websocket.tokenIssueSecret`）取 session token，再调 `/api/channels/{definitions,status,:runtime/configure|stop,weixin/login/start|poll}`；401 自动重试一次。
- **`memory-client/`** — 抽象记忆/Runtime 服务：`http-memory-client.ts`(HTTP `MEMMY_MEMORY_LAYER_URL`，含 `retry`/`errors`/`memory-layer-endpoints`) 与 `memos-sqlite-memory-client.ts`(只读直读 Memory SQLite)；`createDefaultMemoryClient` 优先 HTTP、回退 SQLite、都不可用抛错（不静默展示假数据）。
- **`skill-writer/`** — 把 Memmy Skill/插件写入各 Agent 配置；`SkillTarget{targetId,resolveRootDirectory(),install/uninstall/isInstalled,installPlugin?/uninstallPlugin?/detectMemoryPluginConflict?}`、`SkillManifest{targetId,content,marker}`；`target-registry.ts` 按 `targetId` 注册；各 Agent 具体目标在 `claude-code/codex/cursor/hermes/openclaw/opencode/workbuddy/` 子目录，加 `hook-command.ts`/`memmy-runtime-config.ts`/`skill-directory.ts` 助手；在 `services/index.ts` 组装 7 个 target，供 `SkillDistributionService` 消费。

## 5.3.4 基础设施（持久化）

- **`app-state-store/`** — 主 SQLite 库。`db.ts openDatabase` 打开 `DatabaseSync`（`MEMMY_APP_DB_PATH` 或平台默认，macOS `~/Library/Application Support/Memmy/app.sqlite`），`PRAGMA foreign_keys=ON; journal_mode=WAL; synchronous=NORMAL; busy_timeout=5000`。`createAppStateStore` 跑迁移、`schema-finalizer`、捕获 legacy state，组合仓储：`bootstrap/modelConfig/accountSession/agentSources/idempotency/composioMachineToken/byokTokenUsage/deviceIdentity` + `secretStore`(SQLite) + `localDataStore`(FS)。多账户隔离由迁移 `0008-account-isolation.sql` 引入（详见 [06](../06-data-model.md)）。
- **`agent-source-store/`** — `createAgentSourceRepository(db)` 操作 `account_agent_sources`/`watermarks`/`conversation_checkpoints`/`account_ingestion_seen`，scope 到合成账户 uuid `"local-agent-sources"`（`INSERT OR IGNORE` 保证 FK）；提供 list/upsert/remove/setStatus/setLastScannedAt、水位 get/upsert、检查点、`hasSeen/markSeen` 去重。
- **`idempotency-store/`** — `createIdempotencyStore(db,{getActiveUuid})`，按活跃账户 uuid 操作 `account_idempotency_keys`(`lookup/save/purgeBefore`)，无活跃账户抛 `unauthorized`。
- **`agent-source-scan-journal/`** — `createAgentSourceScanJournal(db)` 持久化**可续跑扫描状态**（jobs/source_state/messages/results），`writeResume` 事务化（BEGIN/COMMIT/ROLLBACK）。
- **`cli-binary/`** — `resolveDefaultRuntimeConfigPath()`→`~/.memmy/runtime.json`；`writeRuntimeConfigFile` 原子写（temp+rename, `chmod 0600`, dir `0700`）；`installer.ts` 处理 CLI 安装/符号链接 `~/.local/bin/memmy`。
- **`memmy-config/`** — 读写 `~/.memmy/config.yaml`；`MemmyConfigWriter` 把账号/BYOK 模型配置投影进 YAML（`agents.defaults`/`providers[<mapped>]`/`memmyMemory.profiles.{account,byok}`/`tools.imageGeneration.profiles`/`channels`/`tools.mcpServers`）；`mapModelProtocol` 映射本地协议名到运行时 provider/apiType；`readRuntimeMemmyConfigState` 给出启动状态（`missing/empty/invalid_yaml/no_model_config/conflict/valid_account/valid_byok`）。

## 5.3.5 服务层（应用/领域）

`src/services/index.ts createBackendServices` 在仓储+出站客户端上装配约 20 个服务，要点：

- **`agent-source-service.ts`** — 扫描编排器：`list/scanAll/scanOne/collectOne/collectAll/ingestCollected/processImportSummaries/addManual/importManaged/syncManaged/updateManaged/remove/installSkill/uninstallSkill/installPlugin/uninstallPlugin/detectMemoryPluginConflicts`；驱动 `SourceRegistry`→`IngestionService`→`MemoryClient`，管水位/检查点，强制 `getScanPermission()`，发 `AgentSourceLifecycleAnalytics`；调参 `INITIAL_GLOBAL_MEMORY_LIMIT=1000`、`IMPORT_WORKER_TIMEOUT_MS=600_000`。
- **`agent-source-scan-runner.ts`+`agent-source-scan-worker.ts`** — 扫描管线状态机（`ScanResumeState` 内存态 / `ScanResumeStateReference` SQLite 态），阶段 `add→summarize`。
- **`agent-source-auto-scan-service.ts`/`-auto-inject-service.ts`** — 周期后台扫描与自动注入。
- **`onboarding-insight-service.ts`** — 首登洞察报告：采样≤12 会话文件/24 查询，正则分类（topic/problem/decision/action），可选 LLM（account `memmy_account/agent_chat` 或 BYOK），≤2000 token，SSE 流式。
- **`token-quota-service.ts`** — 代理云配额 eligibility/request（按活跃账户 uuid）。
- **`byok-token-usage-service.ts`** — `ByokTokenUsageRepository`(`recordEvent/getSummary`) 薄封装。
- **`account-service.ts`** — send-code（60s 节流持久化）、verify-code（云登录→写 config.yaml→reloadConfig→upsert session）、profile/guide/logout/session。
- 其余：`bootstrap-service`/`app-config-service`(+`model-config-tester`)/`integration-service`(Composio)/`channel-service`/`session-service`+`turn-service`(幂等包装)/`search-service`/`memory-detail-service`/`panel-service`/`local-data-service`/`asr-service`/`ingestion-service`/`skill-distribution-service`/`progress-bus`(EventEmitter `scan_progress/scan_completed`)/`runtime-config-sync-service`/`desktop-install-state-service`。

## 5.3.6 权限与埋点

- **`permission/permission-manager.ts`** `createPermissionManager({appStateStore,runtimeToken?})`：生成/持久化运行时 token；`getScanPermission` 读 onboarding（可 `setScanPermission` 覆盖）；`canScanAgentSource` 需 `scan_only|scan_and_write_skill`；`canWriteAgentSkill` 需 `scan_and_write_skill`；`canDetectAgentSources/canSearchMemory` 恒真；`verifyRuntimeToken` 用 `timingSafeEqual`。
- **`analytics/`** — `analytics-transport.ts`(GA4 Measurement Protocol 事件队列，带 env/edition/user-mode)、`ga4-client.ts`(`sendGa4Events`/`resolveGa4Config`)、`agent-source-analytics.ts`(生命周期事件：安装类型、错误经 `errorCodeFromUnknown`、登录用户 id)。

> 设计评价：**优点**：教科书式六边形、领域逻辑独立于框架、多账户隔离、幂等贯穿、权限分级（扫描/写 Skill）、统一错误信封。**潜在改进**：服务数量多（~20）且 `agent-source-service` 承担过多职责，可按"发现/扫描/入库/Skill 分发"再拆；`node:sqlite` 与 Memory 的 `better-sqlite3` 并存增加心智负担（有意隔离，见 ADR）。

---

> ← [走读索引](./index.md) ｜ 上一个 → [02 Agent Runtime](./02-agent-runtime.md) ｜ 下一个 → [04 Agent 来源](./04-agent-sources.md)

---

# ❤️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../../res/promotion.jpg)