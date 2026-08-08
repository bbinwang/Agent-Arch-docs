# 第 5 章 核心代码深度走读

> 本章按重要性逐文件、逐函数、逐模块详细讲解 WeKnora 的核心代码，涵盖功能、参数、返回值、核心逻辑、设计模式、潜在问题与改进建议。基于 1,492 个 Go 文件、37.5 万行代码的深度分析。

---

## 5.1 入口与启动（cmd/server/）

### 5.1.1 `cmd/server/main.go`

**功能**：应用程序唯一入口，负责 Gin 模式设置、容器构建、启动钩子、HTTP Server 创建、信号处理、优雅关闭。

**核心逻辑**：

```go
func main() {
    // 1. Gin 模式设置
    if os.Getenv("GIN_MODE") == "release" {
        gin.SetMode(gin.ReleaseMode)
    }

    // 2. 抑制 Gin 路由注册垃圾输出（~150 行/路由）
    runtime.SilenceGinRouteSpam()

    // 3. 打印环境变量 Banner（敏感值掩码）
    runtime.LogStartupEnv(context.Background())

    // 4. 记录启动时间（用于 Uptime 计算）
    runtime.MarkServerStarted()

    // 5. 构建 DI 容器（~150 个 provider）
    c := container.BuildContainer(runtime.GetContainer())

    // 6. 启动钩子（最佳努力，不中止启动）
    runStartupBootstrap(c)

    // 7. 创建 HTTP Server + 带重试的 Listener
    // 8. 信号处理（SIGINT/SIGTERM）+ 优雅关闭
}
```

**设计决策**：
- `bootstrap` 是**最佳努力**（best-effort）的——任何失败只警告不中止。理由：运维人员打错一个环境变量不应导致整个服务无法启动，应能从运行中的实例修复。
- `bootstrapEnvVar = "WEKNORA_BOOTSTRAP_SYSTEM_ADMIN_EMAIL"`：通过环境变量指定首个系统管理员邮箱，零摩擦初始化。**幂等**——已有管理员后自动失效，防止 UI 撤销被静默恢复。
- 优雅关闭顺序：`listener.Close()` → `server.Shutdown(ctx)` → 清理回调（LIFO）。第二信号强制 `server.Close()`。

**关键函数**：
- `listenWithRetry(addr, 10, 300ms)`：绑定失败时重试 10 次（端口被占用场景）
- `bootstrapSystemAdmin(ctx, userSvc, email)`：仅在无管理员时提升，不创建用户

### 5.1.2 `cmd/server/bootstrap.go`

**功能**：启动后一次性钩子。当前两个职责：
1. **API Key 哈希回填**：`BackfillMissingKeyHashes()` 修复迁移 000065 的占位行，每启动执行一次，全部回填后短路（EXISTS 检查）。
2. **系统管理员提升**：见上文。

**设计模式**：`dig.Invoke` 延迟解析——容器未注册 UserService 时仅警告不崩溃。

### 5.1.3 `internal/runtime/startup.go`

**功能**：
- `SilenceGinRouteSpam()`：替换 `gin.DebugPrintRouteFunc`，用原子计数器替代逐路由打印，启动后一次性打印 `"[gin] registered N routes"`。
- `LogStartupEnv(ctx)`：打印精选环境变量（安全/运行时/DB/缓存/存储/外部），敏感值显示为 `"set (N chars)"`，未设置显示 `"<unset>"`。附带 footgun 警告（如 `SYSTEM_AES_KEY` ≠32 字节时加密静默禁用）。

**安全考量**：环境变量日志中 `SYSTEM_AES_KEY`、`REDIS_PASSWORD` 等敏感值被掩码，防止日志泄露。

---

## 5.2 DI 容器（internal/container/）

### 5.2.1 `container/container.go`（1674 行）

**功能**：整个系统的"组装图"，`BuildContainer` 函数注册 ~150 个 provider。

**分层注册顺序**：

```
1. ResourceCleaner（资源清理器）
2. 核心基础设施
   - config.LoadConfig（配置加载）
   - initLangfuse（Langfuse 追踪）
   - initDatabase（数据库连接 + 迁移 + 数据修复）
   - initFileService（文件服务）
   - initRedisClient（Redis 客户端）
   - initAntsPool（goroutine 池）
3. 检索引擎注册表
4. 外部服务客户端
   - initDocReaderClient（文档解析 gRPC/HTTP）
   - docparser.NewImageResolver（图片解析器）
   - initOllamaService（Ollama 本地服务）
   - initNeo4jClient（Neo4j 图谱数据库）
   - stream.NewStreamManager（流管理器）
   - NewDuckDB（DuckDB 嵌入式分析）
5. Repository 层（~35 个）
6. MCP 管理器
7. Service 层（~35 个）
8. Web Search 注册表（9 个搜索引擎）
9. Agent/Session 服务
10. Task Enqueuer（asynq 或 sync）
11. Connector Registry（4 个数据源连接器）
12. DataSource Scheduler
13. Chat Pipeline 插件（~12 个）
14. HTTP Handler（~30 个）
15. IM 适配器（9 个）
16. Router + Asynq Server
17. Wiki 任务恢复
```

**关键设计**：

**Lite vs 分布式分支**：
```go
redisAvailable := os.Getenv("REDIS_ADDR") != ""
if redisAvailable {
    // asynq 任务队列 + Redis 流 + 分布式并发治理
} else {
    // SyncTaskExecutor（内存 goroutine）+ 内存流 + 本地并发治理
}
```

**数据库迁移后数据修复**（`initDatabase` 内）：
- `resolveStorageProviderPending`：将 `__pending_env__` 占位符替换为真实 `STORAGE_TYPE`
- `migrateLegacyStorageBackends`：从旧配置回填 `storage_backends` 表（幂等，每次启动刷新 env 别名）
- `syncSequences`：PostgreSQL seq_id 安全同步

**Web Search 注册表**：注册 9 个搜索引擎（DuckDuckGo、Google、Bing、Tavily、Ollama、Baidu、SearXNG、Keenable、Zhipu）。

**连接器注册表**：注册 4 个数据源连接器（feishu/lark/notion/yuque/rss），错误通过 `errors.Join` 聚合。

**Chat Pipeline 插件**：通过 `Invoke` 注册 12 个插件——search、rerank、web-fetch、merge、data-analysis、into-chat-message、chat-completion、chat-completion-stream、filter-top-k、query-understand、load-history、extract-entity、search-entity、search-parallel、wiki-boost。

**模型并发治理**：`defaultModelMaxConcurrency = 32`，值 ≤0 禁用治理器。

### 5.2.2 `container/engine_factory.go`（323 行）

**功能**：运行时动态构建 `RetrieveEngineService`，注入 `VectorStoreService` 使向量库通过 API 创建后无需重启即可使用。

**支持的引擎类型**（10 个）：OpenSearch、PostgreSQL、SQLite、Elasticsearch（v7/v8 自动检测）、Qdrant、Milvus、Weaviate、Doris、Tencent VectorDB。

**设计模式**：Strategy/Factory —— 每个引擎类型是独立构造函数，新增引擎只需添加 case + 函数。

**特殊处理**：
- PostgreSQL 仅支持 `UseDefaultConnection=true`（拒绝自定义连接）
- Doris 使用双端口：MySQL 协议（默认 9030）读写 + HTTP（默认 8030）Stream Load
- OpenSearch env-store ID 映射为 `""`（env 集群共享，无前缀）

### 5.2.3 `container/cleanup.go`（86 行）

**功能**：优雅关闭时按**反向注册顺序**（LIFO）执行清理回调，容忍单个失败。

```go
func (c *ResourceCleaner) Cleanup(ctx context.Context) (errs []error) {
    // 反向遍历，sync.Mutex 保护
    // 每个 cleanup 独立执行，错误累积不中止
}
```

### 5.2.4 `container/recover_pending_wiki_tasks.go`（109 行）

**功能**：Wiki 触发器是易失的（Lite 模式重启丢失，Redis 中断丢失），启动时重新武装。

**核心逻辑**：
1. 删除 `task_pending_ops` 中 KB 已不存在的行（子查询 activeKnowledgeBase，错误时故障关闭）
2. 为每个活跃 KB 的每个 pending lane 入队一个 `WikiIngestPayload` 触发器
3. Finalize 任务使用确定性 `TaskID("wiki-finalize-"+scopeID)` 使同时副本合并为一个触发器

### 5.2.5 `container/reset_pending_tasks.go`（188 行）

**功能**：重启后将卡住的 "processing" 状态重置为失败。

**Lite vs 分布式差异**：
- **Lite**：直接重置卡住的 knowledge/summary/sync（另一个副本不可能在运行）
- **分布式**：不重置 knowledge/summary（另一个副本可能正在运行），仅重置同步日志（30 分钟过期截止）

---

## 5.3 配置系统（internal/config/）

### 5.3.1 `config/config.go`（1104 行）

**功能**：定义完整配置类型层次 + 加载 YAML + 环境变量覆盖 + 模板回填 + 验证。

**根配置结构**：
```go
type Config struct {
    Conversation    *ConversationConfig
    Server          *ServerConfig
    KnowledgeBase   *KnowledgeBaseConfig
    Tenant          *TenantConfig
    Auth            *AuthConfig
    Audit           *AuditConfig
    OIDCAuth        *OIDCAuthConfig
    Models          []ModelConfig
    VectorDatabase  *VectorDatabaseConfig
    DocReader       *DocReaderConfig
    StreamManager   *StreamManagerConfig
    ExtractManager  *ExtractManagerConfig
    WebSearch       *WebSearchConfig
    PromptTemplates *PromptTemplatesConfig
    IM              *IMConfig
    Agent           *AgentConfig
    FrontendBaseURL string
}
```

**环境变量覆盖**：
- `DISABLE_REGISTRATION=true` → 强制 `invite_only`
- `WEKNORA_TENANT_ENABLE_RBAC` / `MAX_OWNED_PER_USER` / `SELF_SERVICE_CREATION_ENABLED`
- OIDC 字段、Agent 超时、KB 超时、审计保留天数

**验证规则**：OIDC 字段、认证枚举、审计保留 ≥0、Conversation top-k/threshold 范围、chunk size/overlap、server port 1-65535。

**模板回填**：`backfillConversationDefaults()` 从 `prompt_templates/*.yaml` 加载模板文本，填充 `FallbackPrompt`、`RewritePrompt` 等字段。

---

## 5.4 路由与中间件

### 5.4.1 `router/router.go`（2391 行）

**功能**：Gin 引擎构建 + 全局中间件注册 + ~150 个端点注册。

**中间件链**：
```
CORS → RequestID → Language → Logger → Recovery → ErrorHandler →
Auth → FileServing → PresignedRoutes → Langfuse → Audit →
[/api/v1 group] → RBAC Guards + APIKeyGate → Register* → assertAPIKeyPoliciesMatchRoutes
```

**API Key 策略断言**：`assertAPIKeyPoliciesMatchRoutes` 在启动时验证每个声明的策略映射到真实路由，防止策略/路由漂移。

**关键设计**：
- `trustedProxies()`：限制为前置代理网络，防止 XFF 欺骗绕过 embed 限流
- `embedFrameAncestorsMiddleware`：CSP frame-ancestors 头
- `serveFrontendStatic`：Lite 模式下 Go 嵌入服务前端静态资源

### 5.4.2 `router/rbac.go`（631 行）

**功能**：集中管理所有 RBAC 守卫和 API Key 策略构造器。

**双轴授权模型**：
1. **角色守卫**（`Viewer/Contributor/Admin/Owner`）：问"调用者在租户中的角色是什么？"
2. **所有权守卫**（`OwnedKBOrAdmin/OwnedAgentOrAdmin`）：问"调用者是此资源的创建者还是 Admin+？"

**选择指南**（文件头注释是权威参考）：
- 资源有创建者（KB/Agent/Knowledge/Chunk/WikiPage/FAQ/Tag）→ `OwnedXxxOrAdmin`
- 租户级基础设施（Model/VectorStore/IM/WebSearchProvider/DataSource/MCPService）→ `Admin()`
- 创建新资源（POST /knowledge-bases）→ `Contributor()`

**API Key 策略**：能力值类型 + 流畅 `WithCapability` 构建器，默认拒绝未声明路由。

### 5.4.3 `router/task.go`（460 行）

**功能**：Redis/asynq 后台任务基础设施。

**6 个独立 Worker-Pool**：
| 队列 | 用途 | 特点 |
|------|------|------|
| QueueCore | 核心解析 | 主处理队列 |
| QueuePostProcess | 后处理 | VLM/图谱/问答生成 |
| QueueEnrichment | 增强 | 数据增强 |
| QueueMaintenance | 维护 | 复制/删除/移动 |
| QueueShared | 弹性共享 | 可借用给 Core/Enrichment |
| QueueWiki | Wiki 生成 | 硬隔离 |

**死信处理**：`newDeadLetterKnowledgeFailer` 回调在任务耗尽重试后将父 Knowledge 标记为失败（单 UPDATE，8KB 错误上限）。

**重试退避**：`WikiIngestConcurrent` 错误固定 15s 退避，其余 asynq 默认指数退避。

### 5.4.4 `router/task_inspector.go`（1097 行）

**功能**：管理员运行时仪表盘 —— 取消、队列统计、游标分页任务列表、Worker 状态。

**游标分页**：`runtimeTaskCursor` 使用 base64-JSON 续传游标（max 32 anchors, 16KB），基于 LPos/ZRank/ZRank 的锚点偏移。

**两阶段取消**：
1. 排空队列中任务
2. 快照 + 取消活跃任务 → 重新排空已转换任务

**状态感知存储排序**：
- pending/active/archived/completed → 最新优先
- scheduled/retry → 最早优先

### 5.4.5 `router/sync_task.go`（156 行）

**功能**：Lite 模式（无 Redis）的 `TaskEnqueuer` 兼容替代品。

**核心逻辑**：`Enqueue()` 查找 handler → 解析 `ProcessInOpt`（延迟）和 `MaxRetryOpt`（默认 25）→ 启动 goroutine → 延迟后执行 → 线性退避重试（attempt*5s, 上限 30s）。

### 5.4.6 中间件精选

#### `middleware/auth.go` — JWT/API Key 认证

**认证流程**：
1. `noAuthAPI` 白名单检查（前缀 globbing `*` 后缀 + 精确匹配）
2. JWT 优先 → API Key 回退
3. 租户解析（JWT claim → X-Tenant-ID → 首个成员）
4. 角色解析（4 步决策：成员行 → 跨租户超级用户 → 孤儿自愈 → RBAC 标志）
5. 上下文注入

**跨租户切换**：`X-Tenant-ID` 头 + 用户属性 + 集群标志三重验证。

#### `middleware/rbac.go` — RBAC 守卫

**关键设计**：`EnableRBAC=false` 时所有权查找**永不执行**（零额外 DB 成本），仅记录日志放行。API Key 主体始终短路（由 `APIKeyGate` 单独授权）。

#### `middleware/api_key_gate.go` — API Key 路由守卫

**核心数据结构**：
```go
type APIKeyRoutePolicy struct {
    PlatformOnly     bool
    RequireFullAccess bool
    Capabilities      []types.APIKeyCapability
}
```

**默认拒绝**：未声明策略的路由不允许 API Key 访问（故障关闭）。`normalizeRoutePath` 折叠 `//` 去除尾部 `/` 确保路径匹配。

#### `middleware/kb_access.go` — 知识库访问控制

**3 步检查**：自有 → 组织共享（`CheckTenantKBPermission`）→ Agent 共享（只读）。

**有效租户重写**：将 `c.Request.Context()` 重写为携带有效租户 ID，使下游检索命中正确嵌入库。

#### `middleware/logger.go` — 请求/响应日志

**安全特性**：
- `loggerResponseBodyWriter` 上限 10KB（防止 SSE 无限内存）
- `sanitizeBody` 通过大小写不敏感正则脱敏敏感字段
- `sanitizeQuery` 脱敏 OAuth code/state/token
- `readRequestBody` 完全读取 + 重置 body（下游 handler 不受影响）

---

## 5.5 Handler 层精选

### 5.5.1 `handler/auth.go` — 认证处理器

**HTTP 端点**：
| 方法 | 路径 | 功能 |
|------|------|------|
| POST | /auth/register | 公开注册（invite_only 时阻止）|
| POST | /auth/login | Bearer 登录 |
| POST | /auth/refresh | Refresh Token 续期 |
| GET | /auth/me | 当前用户信息 |
| POST | /auth/switch-tenant | 切换租户（重发 Token）|
| GET | /auth/oidc/url | OIDC 授权 URL |
| GET | /auth/oidc/callback | OIDC 回调（code → Token）|

**OIDC 安全**：
- 一次性 nonce 绑定 secure SameSite-Lax cookie
- `encodeOIDCCallbackPayload`/`decodeOIDCState` 使用 HMAC-SHA256 state + nonce cookie
- 服务端 code exchange（非前端）

**注册模式 3 级解析**：DB system_settings → cfg → `self_serve` 默认值。

### 5.5.2 `handler/knowledge.go` — 知识文档处理器

**核心端点**：
| 方法 | 路径 | 功能 |
|------|------|------|
| POST | /knowledge-bases/:id/knowledge/file | 文件上传 |
| POST | /knowledge-bases/:id/knowledge/url | URL 导入（SSRF 校验）|
| POST | /knowledge-bases/:id/knowledge/manual | Markdown 手动创建 |
| GET | /knowledge/:id | 知识详情（含 5 阶段 trace）|
| DELETE | /knowledge/:id | 异步删除 |
| POST | /knowledge/:id/reparse | 异步重解析 |
| POST | /knowledge/move | 跨 KB 移动 |

**安全特性**：
- `requireKBOwnershipOrAdmin` 使用 `EvaluateOwnershipOrRole`
- `resolveKnowledgeAndValidateKBAccess` 双重验证
- `filterKnowledgesByKBAllowSet` API Key 范围过滤

### 5.5.3 `handler/session/qa.go` — 问答处理器（核心）

**`parseQARequest` 流程**：
1. 参数绑定 + 校验
2. SSRF 剥离图片 URL
3. 建议归属校验
4. 解析自有会话
5. 解析共享/自有 Agent
6. 合并 @mentions
7. 处理 base64 图片
8. 并发处理文件附件（音频走 ASR）
9. 解析预上传文档元数据
10. 构建标签/MCP/技能作用域
11. 构造 `qaRequestContext`

**`executeQA` 流程**：
1. 持久化输入栏状态
2. 发射 agent_query
3. 创建用户 + 助手消息
4. 设置 SSE 头
5. 注册完成处理器
6. 异步执行（解析附件 → VLM 分析 → KnowledgeQA/AgentQA）
7. panic 恢复
8. 完成助手消息（Agent 模式用 `WithoutCancel` defer）

### 5.5.4 `handler/session/agent_stream_handler.go` — Agent 流处理

**功能**：订阅 Agent EventBus 所有事件，归一化 `StreamEvent` 写入 StreamManager。

**关键逻辑**：
- 跟踪每事件开始时间
- TTFB（首字节时间）日志
- 知识引用收集
- 答案段处理（含非终端前文的 supersede 处理）
- 完成事件（无答案时发射降级答案）

### 5.5.5 `handler/wiki_page.go` — Wiki 页面处理器

**端点列表**：
| 方法 | 路径 | 功能 |
|------|------|------|
| GET | /wiki/pages | 页面列表 |
| POST | /wiki/pages | 创建页面 |
| GET | /wiki/pages/:slug | 获取页面 |
| PUT | /wiki/pages/:slug | 更新页面 |
| DELETE | /wiki/pages/:slug | 软删除 |
| GET | /wiki/index | 索引（分页目录）|
| GET | /wiki/graph | 知识图谱（overview/ego）|
| GET | /wiki/log | 日志（游标分页）|
| POST | /wiki/rebuild-links | 重建互链 |
| POST | /wiki/lint | 质量检查 |
| POST | /wiki/auto-fix | 自动修复 |

**图谱模式**：overview（全图）/ ego（自我中心，depth 1-3, limit ≤2000）。

---

## 5.6 Application/Service 层核心

### 5.6.1 `service/agent_service.go` — Agent 服务

**功能**：编排 Agent 引擎的创建和配置。

**`CreateAgentEngine` 流程**：
1. 校验配置（`MaxIterations` 限制 [1,100]）
2. 构建工具注册表
3. 注册 MCP 工具（all/selected/none 模式）
4. 解析知识库/文档元数据
5. 解析系统提示
6. 构造 `agent.NewAgentEngine`
7. 附加 VLM 图片描述器 + 技能管理器

**工具注册表 + 能力过滤**：
- 仅当运行时前提存在时注册工具（wiki 工具仅在 wiki-capable KB 在作用域内时注册）
- 白名单策略（`config.AllowedTools`，回退 `tools.DefaultAllowedTools()`）
- 安全网：前置条件缺失的工具被丢弃

### 5.6.2 `service/chat_pipeline/` — 对话管道

**设计模式**：插件式事件驱动管道，`EventManager` 注册插件并构建职责链。

**核心插件**：
| 插件 | 事件 | 功能 |
|------|------|------|
| PluginSearch | SEARCH | 混合检索 + 查询扩展 + 网络搜索回退 |
| PluginRerank | RERANK | 重排 + MMR 多样性 + 回退最低分 |
| PluginMerge | MERGE | 结果融合 + 去重 + 历史注入 + 父分块解析 |
| PluginChatCompletion | CHAT_COMPLETION | LLM 对话生成 |
| PluginChatCompletionStream | CHAT_COMPLETION_STREAM | 流式生成 + 事件路由 |
| PluginQueryUnderstand | QUERY_UNDERSTAND | 查询改写/意图分类 |
| PluginFilterTopK | FILTER_TOP_K | 确定性排序 + top-k |
| PluginIntoChatMessage | INTO_CHAT_MESSAGE | 检索结果渲染为消息 |
| PluginWebFetch | WEB_FETCH | 网页全文抓取 |
| PluginDataAnalysis | DATA_ANALYSIS | DuckDB 数据分析 |
| PluginWikiBoost | WIKI_BOOST | Wiki 提升因子 1.3 |

### 5.6.3 `service/knowledge_process.go`（~3450 行）— 知识处理核心

**`processChunks` 主工作流**：
1. 幂等清理旧分块/索引/图谱
2. 创建分块对象（支持 parent-child）
3. 保存到数据库
4. 构建向量索引（仅启用嵌入时）
5. 检查存储配额
6. 入队多模态 + 后处理任务

**阶段追踪**：`beginStage`/`endStage`/`failStage` + attempt 机制（每次重新处理获得新 attempt，过期任务通过 `attemptSuperseded` 跳过）。

**取消检查点**：`isKnowledgeAborted` 在多个阶段检查（分块写入前后、索引前后），删除状态触发清理，取消状态保留数据。

**摘要生成**：
- 从分块重建文档
- 丰富图片信息
- 长内容采样（头 60%/尾 20%/中 20%）
- LLM 生成摘要

### 5.6.4 `service/wiki_ingest.go`（~117KB）— Wiki 摄入引擎

**设计模式**：
- **Pending-op 队列**：claim/peek/trim + stale-claim recheck
- **Slug 锁**：每 slug 互斥
- **Inflight 槽位预留**：Redis Lua 脚本（`wikiInflightReserveScript`）
- **Finalize 分阶段**：`wikiFinalizeRow`/`wikiFinalizeChange`

**并发控制**：`withSlugLock` + `reserveInflightSlot` 确保同一 slug 不被并发写入。

### 5.6.5 `service/session_qa.go` — Knowledge QA

**`KnowledgeQA` 流程**：
1. 解析知识库
2. 解析聊天模型
3. 构建 `ChatManage`
4. 通过 `EventManager.Trigger` 运行管道

### 5.6.6 `service/session_agent_qa.go` — Agent QA

**`AgentQA` 流程**：
1. 构建 Agent 配置
2. 创建引擎
3. 执行

**作用域应用**：
- `applyPerRequestSkillScope`：每轮技能作用域
- `applyPerRequestMCPScope`：每轮 MCP 作用域
- `intersectPreservingRequestOrder`：保留请求顺序的交集

### 5.6.7 `service/tenant.go` — 租户服务

**`CreateTenant` 流程**：
1. 创建租户记录
2. 创建默认存储后端
3. 添加 Owner 成员
4. 可选创建默认 API Key
5. 设置首个租户为默认

**配额限制**：每用户拥有租户数上限（3 级解析器，TOCTOU 创建后重新计数 + 回滚）。

### 5.6.8 `service/user.go`（~50KB）— 用户服务

**认证方法**：
- `Register`/`Login`/`LoginWithOIDC`
- `GenerateTokens`/`RefreshToken`/`ValidateToken`/`Logout`
- `AdminResetPassword`（哈希 + 撤销所有会话）

**OIDC 安全**：
- 发现拒绝内部端点
- Token 交换阻止重定向到内部 URL
- 错误不回显密钥

---

## 5.7 Agent 引擎深度走读

### 5.7.1 `agent/engine.go` — AgentEngine 核心

**结构体字段**（19 个）：
```go
type AgentEngine struct {
    config               *types.AgentConfig       // Agent 配置
    toolRegistry         *agenttools.ToolRegistry // 工具注册表
    chatModel            chat.Chat                // LLM 聊天接口
    eventBus             *event.EventBus          // 事件总线
    knowledgeBasesInfo   []*KnowledgeBaseInfo      // 绑定知识库信息
    selectedDocs         []*SelectedDocumentInfo   // @提及文档
    pinnedMCPServices    []*PinnedMCPServiceInfo   // @提及 MCP 服务
    pinnedSkills         []*PinnedSkillInfo        // @提及技能
    sessionID            string                    // 会话 ID
    systemPromptTemplate string                    // 系统提示模板
    skillsManager        *skills.Manager           // 技能管理器
    appConfig            *appconfig.Config         // 应用配置
    imageDescriber       ImageDescriberFunc        // VLM 图片描述器
    tokenEstimator       *agenttoken.Estimator     // Token 估算器
    memoryConsolidator   *agentmemory.Consolidator // 记忆整合器
    lastUsage            types.TokenUsage          // 最近一次 Token 用量
    lastSentMsgCount     int                       // 最近一次消息数
    resourceRefs         *llmresource.Registry     // 资源引用别名
    sourceRefs           *llmreference.Registry    // 来源引用别名
}
```

**`Execute` 入口**（line 195）：
```go
func (e *AgentEngine) Execute(ctx, sessionID, messageID, query string, llmContext []chat.Message, imageURLs ...[]string) (*types.AgentState, error) {
    // 1. defer 工具注册表清理
    // 2. 打开顶层 Langfuse Span
    // 3. 初始化 AgentState
    // 4. 构建系统提示 + 消息
    // 5. 构建工具定义
    // 6. 执行 executeLoop
    // 7. 所有退出路径发射完成事件
}
```

**`executeLoop` 外循环**（line 351）：
- 通过 `completionEmitted` bool + `context.WithoutCancel` 保证**恰好一次** `EventAgentComplete` 发射
- 最多 `MaxIterations` 轮（默认 20）
- 每轮委托 `runReActIteration`
- 处理上下文取消（抢救已有工具结果）
- 空响应重试计数器
- 卡循环检测（重复内容）
- 循环结束后未终止 → `handleMaxIterations`

**`runReActIteration` 单轮**（line 457）：
1. 打开轮级 Langfuse Span（defer finish）
2. 管理上下文窗口
3. 调用 LLM（含重试）
4. 检测卡循环（重复内容）
5. 创建 `AgentStep`
6. 处理流中取消
7. 分析响应判断停止条件
8. 执行工具调用
9. 追加工具结果

**设计模式**：
- **无状态引擎**：引擎跨轮次无状态，历史从 DB 重建
- **哨兵枚举**（`iterOutcome`）：显式循环控制
- **延迟清理 + 恰好一次事件发射**
- **依赖注入**（setter 方法）
- **Langfuse Span 嵌套**：agent.execute → agent.round.N → (chat + tools)

### 5.7.2 `agent/act.go` — 行动阶段

**`executeToolCalls`**（line 165）：
- `ParallelToolCalls` 启用且 ≥2 调用 → `executeToolCallsParallel`
- 否则顺序执行

**`executeToolCallsParallel`**（line 190）：
- `errgroup.WithContext` 并发执行所有工具调用
- 互斥锁保护的结果切片（保持原始顺序）
- 最佳努力：失败不取消兄弟任务
- 每个结果发射 `EventAgentToolResult` + `EventAgentTool`

**`runToolCall`**（line 304）核心管道：
1. 规范化 ID
2. 解析/修复 JSON 参数（`RepairJSON`）
3. 发射工具提示
4. 打开 Langfuse Span
5. 构建 `ToolExecContext`（会话、事件总线、用户 ID、审批 ctx、超时）
6. 每工具超时包装
7. 通过注册表执行
8. 完成 Span
9. 记录管道事件

**安全特性**：
- `toolDisplayNames` 内部名 → 中文显示标签
- `toolHintSensitiveArgs`：`database_query` 参数脱敏（原始 SQL）
- `ApprovalCtx` 保持轮级 ctx 不含每工具超时，允许人工审批阻塞更久

### 5.7.3 `agent/think.go` — 思考阶段

**`streamLLMToEventBus`**（line 31）：
1. 编码资源引用（先），再编码来源引用（**顺序重要**——防止 wiki slug UUID 被篡改）
2. 计算提示前缀指纹
3. 流式处理 chunks，按 `ResponseType` 路由：
   - 错误单独捕获（不追加到内容）
   - thinking vs answer 分离
   - 工具调用从别名解码
4. 刷新解码器/扩展器
5. 仅在流错误且无可用内容/工具调用时返回错误

**`streamThinkingToEventBus`**（line 153）：
- 构建 `ChatOptions`（temperature、tools、thinking、parallel tool calls）
- 流式 chunks 路由：
  - tool-call-pending 事件
  - thinking-tool 流
  - `reasoning_content`（DeepSeek 风格）
  - 普通内容（乐观路由到答案区作为前文）
- `ThinkStreamSplitter` 拆分内联 `<think>` 块
- 返回 `ChatResponse` 含 `AnswerStreamed`/`AnswerEventID`

**`callLLMWithRetry`**（line 370）：
- 记录消息摘要（仅尾部）
- 消毒消息
- 调用 `streamThinkingToEventBus`
- 瞬态错误重试（`maxLLMRetries=2`，线性退避）
- 不可恢复失败 + 已有工具结果 → 优雅降级（`streamFinalAnswerToEventBus`）

### 5.7.4 `agent/observe.go` — 观察阶段

**`manageContextWindow`**（line 233）：
- `MaxContextTokens > 0` 时先尝试 LLM 语义摘要压缩
- 回退到 `agenttoken.CompressContext` 确定性裁剪

**`analyzeResponse`**（line 81）停止条件分析：
- Case 0：`content_filter` + 无工具调用 → 终止（发射安全消息）
- Case 1：自然停止（`stop`/`end_turn`/`stop_sequence`）+ 无工具调用 → 终止
- 剥离 `<think>` 块
- 仅当已流式传输时发射 Done 标记，否则发射完整内容 + Done
- 返回 `responseVerdict`（含 `emptyContent` 标志供重试路径）

**`buildRuntimeContextBlock`**（line 233）：
- 每轮 XML 元数据块：`<runtime_context scope="this_turn">`
- 包含 `<current_time>`、`<session>`、`<bound_knowledge_bases>`、`<pinned_documents>`
- `<communication_instruction>`：答案中不使用内部工具名/ID
- `<answer_instruction>`：以纯文本写完整答案后停止使用工具
- **不持久化到历史**（仅当前轮注入）

**`redactHistoryKBResults`**：替换历史 KB 工具结果为"请执行新搜索"标记，强制 Agent 每轮重新检索。

### 5.7.5 `agent/finalize.go` — 终结阶段

**`streamFinalAnswerToEventBus`**（line 27）：
1. 构建新消息数组：系统提示 + 用户轮次 + 所有工具结果（通过 `sourceRefs.ModelOutput`）
2. 检测 markdown 图片 → 追加图片要求
3. 追加最终答案提示（基于检索内容回答、结构化、同语言）
4. 通过 EventBus 流式推送为 `EventAgentFinalAnswer`
5. 设置 `state.FinalAnswer`
6. 发射 Done 标记（如未发射）
7. 剥离残余 `<think>` 块

### 5.7.6 `agent/tools/registry.go` — 工具注册表

**核心设计**：
- **先赢策略**（First-wins）：拒绝重复名称注册，防止工具执行劫持（GHSA-67q9-58v7-32qx 安全公告）
- **字母排序**：`ListTools()`/`GetFunctionDefinitions()` 按名称排序，字节一致（Provider Prompt Cache 友好）
- **输出截断**：防止上下文窗口中毒
- **错误提示**：`toolErrorHint` 常量追加到错误中引导 LLM 重试

**`ExecuteTool` 管道**：
1. 查找工具
2. 类型转换参数（处理 LLM 奇怪行为如 `"true"` vs `true`）
3. JSON Schema 校验
4. 执行
5. 截断大输出
6. 失败时追加错误提示

### 5.7.7 `agent/tools/knowledge_search.go` — 知识检索工具（最复杂）

**`KnowledgeSearchInput`**：
```go
type KnowledgeSearchInput struct {
    Queries         []string // 多查询（查询扩展）
    KnowledgeBaseIDs []string // 目标知识库
}
```

**`Execute` 完整管道**（line 162）：
1. 解析参数
2. 过滤知识库
3. 校验
4. 设置搜索参数（topK、阈值，配置回退）
5. `concurrentSearchByTargets`（并发混合检索）
6. 去重
7. Rerank（模型优先，LLM 回退）
8. MMR 多样性
9. 最终去重
10. 排序
11. 丰富图片信息
12. `formatOutput`（XML 渲染）

**`concurrentSearchByTargets`**（line 427）：
- 按嵌入模型 key 分组目标
- 每个 (模型, 查询) 只计算一次嵌入
- 分离完整 KB（可组合）vs 特定知识目标
- 并发混合搜索

**`rerankResults`**（line 606）：
- 优先使用 Rerank 模型
- 回退到 LLM 重排（批量 15 结果/批，800 字符/段落）

**`applyMMR`**（line 1471）：
- Maximal Marginal Relevance + Jaccard 相似度
- lambda=0.7（相关性 vs 多样性权衡）

**`deduplicateResults`**（line 1022）：
- 多键去重（ID、父分块 ID、知识+索引）
- 内容签名去重

**`compositeScore`**（line 1432）：
- 加权组合：`0.6*model + 0.3*base + 0.1*sourceWeight`
- 位置先验

### 5.7.8 `agent/skills/manager.go` — 技能管理器

**3 级渐进揭示**：
| 级别 | 内容 | 加载时机 |
|------|------|---------|
| Level 1 | 技能元数据（名称+描述） | 始终加载（系统提示中使用）|
| Level 2 | 完整指令 | `read_skill` 时按需加载 |
| Level 3 | 额外资源文件 | `read_skill_file`/`execute_skill_script` 时 |

**安全特性**：
- 白名单访问控制（`allowedSkills`）
- 沙箱隔离脚本执行
- 路径遍历保护（`loader.go` 拒绝 `..` 和绝对路径）

### 5.7.9 `agent/memory/consolidator.go` — 记忆整合器

**`Consolidate` 流程**（line 80）：
1. 保留系统提示 + 当前轮（最后用户消息起）+ 预算内近期历史
2. 将较旧消息总结为 `[Memory Summary]` 系统消息
3. LLM 失败时回退 `rawArchive`（文本转储）

**`findKeepBoundary`**（line 158）：
- 确定保留多少历史消息
- 尊重 tool_call/tool_result 分组（不拆分工具调用对）

### 5.7.10 `agent/token/estimator.go` — Token 估算器

**设计说明**：权威计数来自 API `Usage`，此估算器是补充性的——用于调用间增量估算和首轮回退。`cl100k_base` 是近似（非 OpenAI 模型不同），但足够在正确时机触发压缩。

**常量**：
- `perMessageOverhead = 3`
- `perConversationTail = 3`

### 5.7.11 `agent/token/compress.go` — 确定性压缩

**`CompressContext`**（line 19）：
- 将较旧历史裁剪到 80% 阈值以下
- 保留系统提示 + 当前轮（最后用户消息起）
- 将消息分组为逻辑单元（assistant + 工具结果），从前面移除直到释放足够 Token

**`groupToolMessages`**（line 85）：
- 将 assistant 消息与其后续工具结果分组为一个单元
- 独立消息自成一组
- **确保 tool_call/tool_result 对永不被拆分**

### 5.7.12 `agent/approval/gate.go` — 审批门控

**设计模式**：
- **Waiter/chan 模式** + `sync.Once` 恰好一次交付
- **Redis Pub/Sub 跨实例扇出** + 每 pending 回复通道 + nonce 实现准确 HTTP 语义
- **故障关闭默认**（可通过 `WEKNORA_AGENT_TOOL_APPROVAL_FAIL_OPEN=true` 覆盖）
- **有上限指数退避**订阅者重连

**`RequestAndWait` 流程**（line 309）：
1. 注册 waiter
2. 发射 `EventToolApprovalRequired`
3. select 阻塞（channel/timeout/ctx）
4. 发射 `EventToolApprovalResolved`

**`Resolve` 流程**（line 539）：
1. 本地投递优先
2. `ErrPendingNotFound` + Redis 时跨实例扇出

---

## 5.8 Models 层

### 5.8.1 `models/chat/` — 对话模型

**统一接口**：
```go
type Chat interface {
    Chat(ctx context.Context, messages []Message, opts *ChatOptions) (*ChatResponse, error)
    StreamChat(ctx context.Context, messages []Message, opts *ChatOptions, streamFunc func(chunk *StreamResponse, fullContent string)) error
}
```

**包装器链**（Decorator 模式）：
```
Base Provider → Langfuse Wrapper → Concurrency Wrapper → LLM Debug Wrapper
```

- `langfuse_wrapper.go`：每次 LLM 调用打开 Span，记录 Token 用量
- `concurrency_wrapper.go`：按模型并发控制（信号量）
- `llm_debug_wrapper.go`：调试用，记录请求/响应
- `thinking.go`：思维模式（DeepSeek `reasoning_content`、OpenAI `thinking`）
- `prompt_cache.go`：Provider Prompt Cache 优化
- `sse_reader.go`：SSE 流解析

### 5.8.2 `models/provider/` — 29 个 LLM Provider

**实现统一接口的 Provider**：
OpenAI、Anthropic、DeepSeek、Azure OpenAI、Gemini、Zhipu、Volcengine（豆包）、Qwen（通义千问）、Ollama、NVIDIA、MiniMax、Hunyuan（混元）、Novita AI、SiliconFlow、OpenRouter、Requesty、Moonshot、Longcat、Mimo、Modelscope、Qianfan、Qiniu、GPUStack、LKEAP、WeKnoraCloud、Generic。

### 5.8.3 `models/limiter/governor.go` — 并发治理器

**功能**：按 LLM 模型限制并发请求数，防止 Provider 限流。

**实现**：Redis 分布式信号量（分布式模式）/ 本地信号量（Lite 模式）。

---

## 5.9 IM 适配器

### 5.9.1 `im/service.go` — IM 服务

**职责**：渠道管理 + QA 队列调度 + 流式回复分发。

### 5.9.2 `im/qaqueue.go` — QA 队列

**限制**：
- 每实例 `MaxQueueSize`（默认 50）
- 全局 `GlobalMaxWorkers`（Redis 计数器）
- 每用户 `MaxPerUser`（默认 3）
- 滑动窗口 `RateLimitWindow`/`RateLimitMax`

### 5.9.3 9 个适配器

| 适配器 | 传输协议 | 特殊能力 |
|--------|---------|---------|
| wecom | WebSocket + Webhook | 引用回复、文件下载 |
| feishu | WebSocket + Webhook | 区域（CN/国际）、富文本 |
| slack | WebSocket + Webhook | 区块 Kit |
| telegram | Long Polling + Webhook | 内联键盘 |
| dingtalk | WebSocket + Webhook | 卡片消息 |
| wechat | Long Polling + Webhook | 二维码、加密 |
| qqbot | WebSocket | 频道消息 |
| yunzhijia | WebSocket | 图片消息、Markdown |
| mattermost | WebSocket + Webhook | 斜杠命令 |

---

## 5.10 MCP 子系统

### 5.10.1 `mcp/client.go` — MCP 客户端

**功能**：连接外部 MCP Server，调用工具/读取资源。

### 5.10.2 `mcp/oauth_manager.go` — OAuth 管理器

**功能**：管理 MCP 服务的 OAuth2 客户端注册和授权流程。

### 5.10.3 `mcp/oauth_lifecycle.go` — OAuth 生命周期

**流程**：
1. 生成授权 URL（PKCE + State）
2. 回调接收 Code
3. 交换 Token
4. 加密存储
5. 自动刷新

### 5.10.4 `mcp/oauth_tokenstore.go` — Token 存储

**安全**：AES-256-GCM 加密存储 Access Token + Refresh Token。

---

## 5.11 数据源

### 5.11.1 `datasource/connector.go` — 连接器接口

```go
type Connector interface {
    ValidateConnection(ctx, config) error
    ListResources(ctx, parentID) ([]Resource, error)
    DownloadResource(ctx, resourceID) ([]byte, error)
    GetSyncCursor() (cursor string, err error)
}
```

### 5.11.2 4 个连接器实现

| 连接器 | 数据源 | 特殊能力 |
|--------|--------|---------|
| feishu | 飞书文档 | 大 Wiki 同步、增量同步 |
| notion | Notion 数据库 | 页面/数据库结构 |
| rss | RSS 订阅 | 定时拉取 |
| yuque | 语雀知识库 | 文档空间 |

### 5.11.3 `datasource/httpclient.go` — SSRF 安全 HTTP 客户端

**功能**：所有出站 HTTP 请求经过 SSRF 校验（内网 IP 阻止、重定向链校验、域名白名单）。

---

## 5.12 基础设施

### 5.12.1 `infrastructure/docparser/` — 文档解析

**引擎注册表**：
- `builtin_converter.go`：内置转换器（Markdown/HTML/JSON/EPUB/MHTML）
- `mineru_converter.go`：MinerU 云服务/本地解析
- `paddleocr_vl_converter.go`：PaddleOCR-VL 视觉语言模型 OCR
- `grpc_parser.go`：gRPC 调用 DocReader
- `http_parser.go`：HTTP 调用 DocReader

### 5.12.2 `infrastructure/chunker/` — 分块器

**分块策略**：
- `heading_splitter.go`：标题层级分割（保持文档结构）
- `heuristic_splitter.go`：启发式分割（段落/句子边界）
- `strategy.go`：自适应 3 级 chunking（短/中/长文档自动选择）

### 5.12.3 `infrastructure/web_search/` — 网络搜索

**9 个搜索引擎**：DuckDuckGo、Bing、Google、Tavily、Baidu、Ollama、SearXNG、Keenable、Zhipu。

**统一接口**：
```go
type WebSearchProvider interface {
    Search(ctx context.Context, query string, opts SearchOptions) (*SearchResult, error)
}
```

---

## 5.13 任务队列

### 5.13.1 `router/task.go` — asynq 服务器

**6 个队列的 Worker-Pool**（前述）+ 死信处理 + Langfuse 中间件。

### 5.13.2 `router/sync_task.go` — Lite 执行器

**接口兼容**的内存替代品，handler 注册表 + RWMutex。

---

## 5.14 Types 层

### 5.14.1 `types/interfaces/` — 48 个接口

**核心接口**：
| 接口 | 职责 |
|------|------|
| KnowledgeService | 知识文档 CRUD + 检索 |
| KnowledgeBaseService | 知识库 CRUD + 搜索 |
| SessionService | 会话生命周期 |
| MessageService | 消息持久化 + 搜索 |
| TenantService | 多租户管理 |
| UserService | 用户认证 + 管理 |
| VectorStoreService | 向量库 CRUD + 检索 |
| RetrieveEngineService | 检索引擎抽象 |
| TaskEnqueuer | 任务入队（asynq/sync 双实现）|
| AuditLogService | 审计日志 |
| EmbedChannelService | 嵌入渠道管理 |
| MCPServiceService | MCP 服务管理 |

---

## 5.15 Utils 层

### 5.15.1 `utils/crypto.go` — 加密工具

**AES-256-GCM**：API Key、MCP 凭证、数据源凭证加密。

### 5.15.2 `utils/security.go` — 安全工具

**SSRF 校验**：`ValidateURLForSSRF()` 阻止内网 IP、校验重定向链、域名白名单。

### 5.15.3 `utils/presign.go` — 预签名 URL

**HMAC-SHA256**：生成/验证预签名 URL，用于临时文件访问。

---

## 5.16 潜在问题与改进建议

### 5.16.1 架构决策

| 当前实现 | 潜在问题 | 改进建议 |
|---------|---------|---------|
| 单进程 asynq 6 服务器 | 队列间负载不均 | 动态权重调整 |
| 内存 Token 估算（cl100k_base）| 非 OpenAI 模型不准确 | 按模型加载对应 tokenizer |
| 工具注册表先赢策略 | 无法覆盖内置工具 | 提供覆盖机制（显式声明）|
| Lite 模式无持久化队列 | 重启丢失任务 | 可选 SQLite 队列 |

### 5.16.2 性能瓶颈

| 热点 | 影响 | 优化建议 |
|------|------|---------|
| 知识检索并发嵌入 | 嵌入 API 调用次数多 | 查询级缓存 |
| Wiki 摄入 LLM 调用 | 大量实体/概念生成 | 批量 LLM 调用 |
| SSE 事件序列化 | 高频小事件 | 批量推送 |

### 5.16.3 安全加固

| 当前措施 | 加固建议 |
|---------|---------|
| SSRF 校验 | 增加 DNS rebinding 防护 |
| API Key 加密 | 支持 HSM/KMS 集成 |
| 审计日志 | 增加日志完整性校验 |

---

> **下一章**：[第 6 章 数据模型与数据库设计](./06-data-model.md) — ER 图、表结构、索引、缓存策略。

---

☕️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../res/promotion.jpg)