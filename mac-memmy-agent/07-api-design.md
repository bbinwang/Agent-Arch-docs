# 07 · API 与接口设计

Memmy 暴露四类 API：**Memory Runtime API**（:18960）、**桌面本地 API**（临时端口）、**OpenAI 兼容 API**（:18990）、**Cloud API**（外部）。另有 SSE 事件流与共享 Zod 契约。

## 7.1 Memory Runtime API（`/api/v1/`，:18960）

由 `Memory/src/server/http.ts` 的 `routeRequest` 分发，鉴权 `authenticate()` 接受 bearer/`x-api-key`/`?token=`，`GET /health` 匿名。下列为 OpenAPI 风格摘要。

| Method | Path | 说明 | 请求体要点 | 响应要点 |
| --- | --- | --- | --- | --- |
| GET | `/api/v1/health` | 健康 | — | `{ok,version,storage.backendId,...}` + `API_ROUTES` |
| POST | `/api/v1/sessions/open` | 开会话(幂等) | `{sessionId,...}` | session 元数据 |
| POST | `/api/v1/sessions/:id/close` | 关会话(幂等) | — | ok |
| POST | `/api/v1/turns/start` | 开始回合(召回钩子) | `{turnId,sessionId,query}` | `{injectedContext,sourceMemoryIds}` |
| POST | `/api/v1/turns/:id/complete` | 完成回合(采集钩子) | `{answer,toolCalls,results,...}` | 写入结果 |
| POST | `/api/v1/memory/search` | 检索 | `{query,mode,limit,...}` | `{hits,sourceMemoryIds,...}` |
| POST | `/api/v1/memory/add` | 写记忆(幂等) | `{memoryValue,layer?,...}` | memory id |
| POST | `/api/v1/memory/processing/status` | 处理状态 | `{ids?}` | 状态映射 |
| POST | `/api/v1/memory/:id/processing/retry` | 重试处理 | — | ok |
| GET | `/api/v1/memory/:id` | 取单条 | — | memory |
| DELETE | `/api/v1/memory/:id` | 删记忆(幂等) | — | ok |
| POST | `/api/v1/worker/run` | 跑一轮 Worker | `{limit}` | 处理统计 |
| POST | `/api/v1/worker/import-summaries/enqueue` | 入队导入摘要 | — | ok |
| GET | `/api/v1/panel/{overview,analysis,items,tasks}` | 面板只读 | — | 聚合数据 |
| DELETE | `/api/v1/panel/tasks/:id` | 删任务 | — | ok |
| GET | `/api/v1/memory/logs` | 工具审计日志 | — | api_logs |
| GET | `/`、`/viewer` | 记忆面板页 | — | HTML(`viewer/static.ts`) |

> scope 鉴权粒度：`anonymous`/`memory:read`/`memory:write`/`panel:read`/`admin:write`。

**检索模式与召回层（请求 `mode`）：** `turn_start`/`search`→Skill/L2/L1/L3；`tool_driven`/`sub_agent`→L2/L1/L3；`skill_invoke`/`decision_repair`→Skill/L2/L1；`world_model`→L3。新 episode 第一轮做意图门控（任务→全层；询问"是否记得"→不含 L3；闲聊/元命令→跳过）。

## 7.2 桌面本地 API（临时端口）

由 `App/backend` Fastify 提供，绑 `127.0.0.1:0`，端口+token 写 `~/.memmy/runtime.json`。**鉴权**：header `x-memmy-local-token`（`timingSafeEqual`），SSE 允许 `?token=`；Composio MCP 桥用独立 `x-memmy-mcp-token`。**CORS**：仅本机/`file://`/受控来源。请求/响应用 `@memmy/local-api-contracts` zod 校验，错误经 `withErrorEnvelope`。

主要路由组（前缀多为 `/api/`，agent-runtime 组为 `/api/v1/`）：

| 组 | 代表端点 |
| --- | --- |
| account | `POST /api/account/{send-code,verify-code,logout}`、`PATCH /api/account/profile`、`GET /api/account/session` |
| agent-runtime | `/api/v1/{health,admin/reload-config,sessions/*,turns/*,memory/*,panel/*}`（代理 Memory 服务） |
| agent-sources | `GET /api/agent-sources`、`POST /api/agent-sources/scan`(权限)、`.../scan/stop`、`.../manual`、`:sourceId/managed/{import,sync}`、`:sourceId/{skill,plugin}` |
| app-config | `PATCH /api/app/{settings,privacy,scan-preferences,onboarding,improvement-program,skin}`、`PUT/GET/POST /api/app/model-config{,/test}` |
| asr | `POST /api/asr/transcriptions` |
| byok-token-usage | `POST /api/app/byok-token-usage/{events,summary}` |
| channels | `/api/v1/channels/{definitions,status,:provider/connect(长轮询),:provider/disconnect}` |
| composio-mcp | `/mcp/composio`(POST/GET/DELETE, `x-memmy-mcp-token`) |
| integrations | `/api/v1/integrations/{capabilities,:slug/authorize,connections,connections/:id}` |
| local-data | `GET /api/local-data`、`POST .../{reveal,export}`、`DELETE /api/local-data` |
| onboarding-insight | `POST /api/onboarding/insight-report{,/stream}` |
| token-quota | `GET /api/token-quota/eligibility`、`POST /api/token-quota/request` |
| events | `GET /api/events`(SSE 15s 心跳 + 扫描进度) |

## 7.3 OpenAI 兼容 API（:18990，`memmy serve`）

`App/memmy-agent/src/entrypoints/openai-like-api/server.ts`。

| Method | Path | 说明 |
| --- | --- | --- |
| GET | `/health` | `{status:"ok"}` |
| GET | `/v1/models` | `{object:"list",data:[{id:<配置模型>,...}]}` |
| POST | `/v1/chat/completions` | 兼容 OpenAI Chat Completions |

**`/v1/chat/completions` 请求示例：**
```jsonc
// JSON
{ "model": "<配置模型>", "messages":[{"role":"user","content":"..."}], "stream": true }
// 或 multipart/form-data: message, session_id, model, files
```
**约束**：仅支持单条 user 消息；`model` 必须等于配置模型否则拒绝；`stream:true` 走 SSE（`sseChunk` + `[DONE]`）。映射到 `loop.processDirect(text,{sessionKey:"api:<session_id>"|"api:default",channel:"api",chatId:"default",media,onStream,onStreamEnd})`。每会话 `withSessionLock`、每请求 `withTimeout`，空回复重试一次。

## 7.4 Cloud API（外部）

由 `cloud-client` 调用，base 来自 `MEMMY_CLOUD_SERVICE`（默认 `https://memmy-api.memtensor.cn`）。云信封 `{code,message,data}`（`code:0`=成功），业务码：`unauthorized 40100`/`forbidden 40101`/限流 `40300-40400`。带 `lang`、`authorization: Bearer`、`x-memmy-device-id`、`x-agent-region`、`x-memmy-composio-token`。代表端点：
- 账户：`/api/agentUser/{sendEmailVerification,sendPhoneVerification,login,info,quota/*}`
- 模型网关（账号模式）：`/api/agentExternal/v1`（OpenAI 兼容形状，加 `X-Agent-Region`）
- Composio：`/api/composio/{auth-configs,integrations/:slug/authorize,connections,connections/:id,router/execute}`
- ASR：`/api/agentAsr/transcriptions`
- BYOK 用量上报（Agent/Runtime 侧）：`${runtime.json.baseUrl}/api/app/byok-token-usage/events`（header `x-memmy-local-token`）

## 7.5 SSE 事件流

- 本地后端 `GET /api/events`：`app.connected`、`app.heartbeat`(15s)、`agent_source.scan_progress`、`agent_source.scan_completed` 等（`progressBus` 转发）。因 EventSource 不能设头，用 `?token=` 传运行时 token。
- onboarding-insight `.../stream`、token-quota 等也用 SSE 推增量。
- OpenAI 兼容 API 的 `stream:true` 用标准 SSE chunk。

## 7.6 共享契约（`@memmy/local-api-contracts`）

被前端、后端、外壳接口、Agent Runtime 共享，是 API 形状单一真相。要点 schema：
- **运行时**：`RuntimeConfigSchema{baseUrl,min(1)localToken,memory?{baseUrl},agentGateway?{baseUrl,bootstrapSecret?}}`。
- **用户态**：`UserModeSchema(unset|byok|account)`、`OnboardingStepSchema(byok_setup_required|account_auth_required|scan_permission_required|initial_report_required|improvement_program_required|product_tour_required|completed)`、`ScanPermissionSchema`、`ImprovementProgramSchema`、`AppSettingsDtoSchema`、`AppBootstrapResponseSchema`。
- **记忆域**：`MemoryKindSchema(trace|span|policy|world_model|skill)`、`MemoryLayerSchema(L1|L2|L3|Skill)`、`MemoryStatusSchema`、`JobTypeSchema(episode_idle_close|trace_summary|reflection|embedding|l2_association|induction|l3_abstraction|skill_crystallization|skill_trial_resolve|…)`、`InjectedContextSchema`、`RecallHitSchema`。
- **Token 配额**：`RequestTokenQuotaInputSchema(reason≥20字符)`、`TokenQuotaApplyResultSchema{requestId,status}`、`TokenQuotaEligibilitySchema(state,requestCount,maxRequestCount=5,nextAllowedAtEpochMs,...)`。
- **常量**：`CLOUD_SERVICE_ENV_KEY="MEMMY_CLOUD_SERVICE"`、`LEGAL_PAGE_PATH`、托管 Agent 发现标记 `MANAGED_AGENT_DISCOVERY_PENDING_DATA_PATH="memmy-agent://history-discovery-pending"`。
- 桌面 IPC 类型在 `@memmy/desktop-interface`（重导出 `RuntimeConfig` + 声明 `memmy:*` 结果类型）。

## 7.7 认证、限流与版本控制小结

| 维度 | 机制 |
| --- | --- |
| 本地 API 认证 | `x-memmy-local-token`（timingSafeEqual，随机 UUID）；SSE `?token=` |
| MCP 桥认证 | 独立 `x-memmy-mcp-token`（`mmt_<random>`） |
| Memory 认证 | bearer token（`MEMMY_MEMORY_TOKEN`/`MEMORY_SERVICE_TOKEN`） |
| Cloud 认证 | `authorization: Bearer <uuid>` + `x-memmy-device-id` |
| 节流 | 验证码 60s 本地节流；Token 配额 `maxRequestCount=5` + `nextAllowedAtEpochMs`；云侧业务码限流 |
| 版本控制 | Memory API `/api/v1/` 前缀；契约用 zod schema 演进；`AppBootstrapResponseSchema`/`health.version` 暴露版本 |

---

> 上一节 ← [06 数据模型](./06-data-model.md) ｜ 下一节 → [08 部署、运维与基础设施](./08-deployment-ops.md)

---

☕️ 制作不易，请我喝咖啡☕️关注我➕