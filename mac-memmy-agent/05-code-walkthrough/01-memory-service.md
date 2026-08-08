# 05.1 · Memory 服务代码走读

包：`@memmy/memory`（`Memory/`）。HTTP 服务默认 `:18960`，数据 `~/.memmy/memory-service/memory.sqlite`。本章按"启动 → 路由 → 编排 → 检索 → 写入与 Worker → 向量 → Schema"顺序走读关键文件。

## 5.1.1 启动与路由

### `Memory/src/server/index.ts` — 服务入口
`main(argv)`：`loadCloudServiceEnv()` 注入 env → 解析 serve 参数 → 加载配置 → 创建存储后端与 `MemoryService` → `listenMemoryHttpServer`。端口解析优先级（19–23 行）：
```ts
const host = options.host ?? process.env.MEMMY_MEMORY_HOST ?? process.env.MEMORY_SERVICE_HOST ?? "127.0.0.1";
const port = options.port ?? numberEnv("MEMMY_MEMORY_PORT") ?? numberEnv("MEMORY_SERVICE_PORT") ?? 18960;
```
启动前还会获取 `<sqlitePath>.server.lock` 文件锁（防止两个服务同库），并把解析出的 endpoint 写回 YAML 配置（`writeCurrentEndpoint`）。**host/port 仅在此处定义，不在 `config/index.ts`。**

### `Memory/src/server/http.ts` — HTTP 层
用原生 `node:http` 的 `createServer`，**无任何 Web 框架**。路由由手写 `routeRequest()`（363 行）分发，`API_ROUTES` 常量（35–57 行）声明所有路由的 method/path/handler/scope。所有业务路径前缀 `/api/v1/`。鉴权 `authenticate()`（928 行）接受 bearer token / `x-api-key` / `?token=`；`GET /health` 恒为匿名。`createAutoWorkerDrain`（197 行）在 `listening` 后启动后台排空循环，并在 `GET /health` 与每次写入/turn-complete 后 `autoWorker.schedule()`，每轮最多 40 个 cycle，优先 `trace_summary`/`import_summary`（批 20）再标准任务（批 100）。

核心路由表（method / path / service 方法 / scope）：

| Method | Path | 方法 | scope |
| --- | --- | --- | --- |
| GET | `/health` | `health(API_ROUTES)` | anonymous |
| POST | `/admin/reload-config` | `reloadConfig` | admin:write |
| POST | `/admin/shutdown` | 触发 `onShutdownRequested` | admin:write |
| POST | `/sessions/open` · `/sessions/:id/close` | `openSession`/`closeSession`（幂等） | write |
| POST | `/turns/start` · `/turns/:id/complete` | `startTurn`（召回钩子）/ `completeTurn`（采集钩子） | read/write |
| POST | `/memory/search` | `search` | read |
| POST | `/memory/add` | `addMemory` | write |
| POST/GET/DELETE | `/memory/:id`、`/memory/processing/*` | `get/delete/retry/status` | read/write |
| POST | `/worker/run` · `/worker/import-summaries/enqueue` | `runWorkerOnce`/`enqueuePendingImportSummaries` | write |
| GET | `/panel/{overview,analysis,items,tasks}` | 面板只读 | panel:read |
| GET | `/memory/logs` | `apiLogs` | panel:read |

> 设计要点：会话/回合/记忆接口多为**幂等**；scope 用作鉴权粒度；`/admin/shutdown` 返回 `{accepted:true}` 后异步触发关闭。

### `Memory/src/cli/index.ts` + `commands.ts` — CLI
`memmy-memory` 是**纯 HTTP 客户端**：`serve` 命令（231–246 行）明确不启动服务，只打印连接方式。命令含 `init/install/health/reload-config/session/turn/search/add/get/delete/raw`。`raw <method> <path>` 是通用逃生口（如 `worker:run` 脚本即 `raw POST /worker/run`）。

## 5.1.2 编排核心 `MemoryService`

`Memory/src/service/memory-service.ts`（~2380 行）的 `MemoryService` 类（227 行）是总编排。构造器（249 行）装配：`Repositories`（存储）、两个 `LlmClient`（`llm`=记忆摘要、`skillLlm`=记忆演化，均由 `createConfiguredMemoryLlm` 构建）、`Embedder`，以及子服务 `ImportJobProcessor`/`RetrievalService`/`WorkerRunner`/`EvolutionJobProcessor`/`FeedbackExperienceService`/`EmbeddingJobProcessor`/`SkillTrialResolver` 与只读模型（`EpisodeReadModel`/`PanelReadModel`/`SkillReadModel`）。演化任务处理器在 269–291 行注册（`induceL2`/`materializeNegativeExperience`/`abstractL3`/`crystallizeSkill`/`associateL2`/`splitBigTurn`/`applyReward`/`reflectTrace`/`resolveSkillTrial`）。

- **写入**：`addMemory`（916–928 行）委托给 `ImportJobProcessor.addMemory`（`import/import-job-processor.ts` 113/116 行）——持久化记忆行后入队 `import_summary`（182 行）或 `trace_summary`。手动 `addMemory` 与采集回合走同一"摘要+向量化"管线。
- **检索**：`search`/`startTurn` 委托 `RetrievalService`。
- **Worker**：`runWorkerOnce(limit, …)` 供 HTTP `/worker/run` 与自动排空调用。

## 5.1.3 混合检索 `RetrievalService`

`Memory/src/service/retrieval/retrieval-service.ts`，`search`（1198 行）流水线：
1. **查询改写**（可选）：evolution 模型生成 3 个互补查询，各独立召回后用独立 RRF 合并（`QUERY_REWRITE_SYSTEM_PROMPT`，98 行），默认关闭。
2. **查询提取**（`RETRIEVAL_QUERY_EXTRACT_PROMPT`）：失败回退原始请求+规则关键词，不中断。
3. **候选池**（`indexed-candidate-pool.ts` `indexedRetrievalCandidatePool`，60 行）：并行跑 `searchVectorIds`（vec_summary/vec_action/vec）、`searchFtsIds`（FTS5）、`searchPatternIds`（LIKE）、`searchStructuralIds`（仅 L1 结构片段）。
4. **融合**（`algorithm/plugin-algorithms.ts` `retrievePluginMemories`，4024 行）：每候选取命中通道最高分 + 层级 bonus + RRF bonus；相对阈值地板；episode 聚合；**MMR**（`mmrSelect` 5319 行，默认 `weightCosine:0.6, weightPriority:0.4, mmrLambda:0.7`）。
5. **LLM 重排过滤**（`filterRecallHits` 1462 行，`RETRIEVAL_FILTER_PROMPT`）：默认开，候选≥2 才调，最多留 8，失败回退保 6，可全量丢弃。
6. **注入**（`buildInjectedContext`）：按 L1→L2→L3→Skill→decision guidance 渲染分层 Markdown 记忆包，普通片段≤640 字符，Skill 默认 summary 模式≤200 字符，明确标注"历史记忆需核对"。

> 详细参数与调优见 [12 关键算法](../12-algorithms-testing.md) 与官方 `docs/cn/memory/overview.mdx`。

## 5.1.4 写入与 Worker

- `Memory/src/service/import/import-job-processor.ts` — `addMemory` 持久化 + 入队；导入 L1 先进 `summary_queued`。
- `Memory/src/service/evolution/span-pipeline.ts` — `summarizeTraceForCapture`（630 行）用 `CAPTURE_SUMMARY_SYSTEM_PROMPT` 调 LLM（`MEMORY_SUMMARY_MAX_TOKENS=512`），主 `llm` 失败回退 `skillLlm`。
- `Memory/src/service/embedding/embedding-job-processor.ts` — `summarizeCapturedTrace`/`summarizeImportedTrace`（213/251 行）在 Worker 作业内执行摘要+向量化。
- `Memory/src/service/worker/worker-runner.ts` — `runWorkerOnce`（269 行）：`leaseQueuedJobs` 租约 `evolution_jobs`、`runEmbeddingRetryOnce`、并发批处理（`SUMMARY_WORKER_CONCURRENCY=4`，embedding 按角色批）。`reconcileWorkerStartup`（147 行）重启后回收中断/租约任务并修复卡住的 `memory_processing_state`。
- 作业类型（`worker/job-handlers.ts` + `evolution/evolution-job-processor.ts`）：处理态 `trace_summary`/`import_summary`/`embedding`；演化态 `induceL2`/`abstractL3`/`crystallizeSkill` 等；失败用独立 `embedding_retry_queue` 退避重试。**无独立 Worker 进程**，`worker` CLI 即 `raw POST /worker/run`，循环也在服务进程内同步跑。

## 5.1.5 向量与 Embedding

- `Memory/src/model/embedder.ts` — `HttpEmbedder`（48 行）：远程 provider（`openai_compatible`/`gemini`/`cohere`/`voyage`/`mistral`）+ 本地（`@huggingface/transformers`，默认 `Xenova/all-MiniLM-L6-v2`，q8/cpu，缓存 `~/.memmy/memory-service/model-cache`）。进程内 `Map` 缓存按 `stableHash(provider:model:role:text)` 去重，可选 L2 归一。
- `Memory/src/storage/sqlite-vec-store.ts` — `SqliteVecStore`（sqlite-vec 0.1.9）。每条记忆最多 3 向量；L1 用 `vec_summary`+`vec_action`，其他层用 `vec`。按维度**惰性建虚拟表** `memory_vec_<dim> USING vec0(embedding float[<dim>] distance_metric=cosine)`。`search`（160 行）限定同维度候选：
  ```sql
  SELECT rowid, distance FROM memory_vec_<dim>
    WHERE embedding MATCH ? AND k = ?
      AND rowid IN (SELECT CAST(value AS INTEGER) FROM json_each(?))
    ORDER BY distance
  ```
  分数 = `1 - distance`（cosine）。Float32↔Buffer 序列化。本地检索先从相应层/字段最近 2000 条向量建搜索窗口再取 Top K（2000 为固定常量，不在 config）。

## 5.1.6 Schema 与 DB

`Memory/src/storage/schema.ts`（`SCHEMA_VERSION=4`，迁移 `004_memory_processing_state`）是唯一真相源。`migrate(db)`（456 行）在事务内幂等执行所有 `statements[]`、记入 `schema_migrations`，升级时跑 `backfillMemoryProcessingState` + `removeLegacyProcessingMetadata`。`Memory/src/storage/db.ts` 的 `MemoryDb`：加载 `sqlite-vec` 扩展，设 `WAL`/`synchronous=NORMAL`/`foreign_keys=ON`/`busy_timeout=5000`，迁移前 `VACUUM INTO` 备份。主要表与索引详见 [06 数据模型](../06-data-model.md)。

> 设计评价：Memory 服务是整个系统技术含量最高的部分。**优点**：混合检索（向量+FTS+pattern+结构）+ RRF + MMR + LLM 过滤的四阶管线、四层记忆与 RL 式演化、向量按维度分表规避 schema 迁移、Worker 租约+死信保证可靠、进程内自动排空避免运维负担。**潜在改进**：2000 条窗口常量不可配；演化参数众多、调参门槛高；无独立 Worker 进程意味着重活会挤占 HTTP 线程（靠批处理与并发上限缓解）。

---

> ← [走读索引](./index.md) ｜ 下一个子系统 → [02 Agent Runtime](./02-agent-runtime.md)

---

# ❤️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../../res/promotion.jpg)