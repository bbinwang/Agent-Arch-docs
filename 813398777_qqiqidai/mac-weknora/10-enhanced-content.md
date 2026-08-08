# 第 10 章 增强内容

> 本章包含五个增强部分：核心组件代码走读文档、开发者上手指南、架构决策记录（ADR）、关键算法分析、测试策略。

---

## 10.1 核心组件独立代码走读文档

### 10.1.1 Agent 引擎代码走读

#### 组件概览

Agent 引擎是 WeKnora 最复杂的核心组件，采用 ReAct（Reasoning + Acting）模式，实现多步推理和工具调用。

#### 核心文件关系

```
engine.go ── 引擎主结构 + Execute 入口 + executeLoop 外循环
    ├── think.go ── Think 阶段（LLM 流式调用）
    ├── act.go ── Act 阶段（工具执行）
    ├── observe.go ── Observe 阶段（响应分析 + 上下文管理）
    ├── finalize.go ── 最终答案生成
    ├── prompts.go ── 系统提示构建
    ├── prompts_wiki.go ── Wiki 提示模板
    ├── tools/ ── 工具注册表 + 67 个工具实现
    ├── skills/ ── 技能系统（3 级渐进揭示）
    ├── memory/ ── LLM 记忆整合器
    ├── token/ ── Token 估算器 + 压缩器
    └── approval/ ── 人工审批门控
```

#### 执行流程详解

**第 1 步：引擎构建**（`NewAgentEngine`）
- 注入配置、聊天模型、工具注册表、事件总线
- 初始化 Token 估算器（`cl100k_base` BPE 编码）
- 初始化资源引用和来源引用注册表（别名编码/解码）
- 若配置 `MaxContextTokens > 0`，创建记忆整合器

**第 2 步：执行入口**（`Execute`）
- 打开顶层 Langfuse Span（`agent.execute`）
- 初始化 `AgentState`（IsComplete=false, Steps=[]）
- 构建系统提示（`buildSystemPrompt`）—— 选择模板（自定义/纯 Agent/Progressive RAG），渲染占位符，追加技能元数据
- 构建消息数组（系统提示 + 历史 + 当前用户消息）
- 构建工具定义（`buildToolsForLLM`）
- 进入 `executeLoop`

**第 3 步：ReAct 循环**（`executeLoop`）
- 最多 `MaxIterations` 轮（默认 20）
- 每轮调用 `runReActIteration`：
  1. **Think**：`callLLMWithRetry` → `streamThinkingToEventBus` → `streamLLMToEventBus`
  2. **Analyze**：`analyzeResponse` 判断停止条件
  3. **Act**：`executeToolCalls`（并行/串行）
  4. **Observe**：`appendToolResults` + `manageContextWindow`
- 终止条件：自然停止、最大迭代、上下文取消
- 循环结束后未终止 → `handleMaxIterations`

**第 4 步：最终答案**（`streamFinalAnswerToEventBus`）
- 基于所有工具结果生成最终答案
- 流式推送为 `EventAgentFinalAnswer`
- 发射 `EventAgentComplete`

#### 关键设计决策

| 决策 | 理由 |
|------|------|
| 引擎无状态 | 水平扩展友好，故障恢复简单 |
| 历史从 DB 重建 | 单一数据源，无缓存一致性问题 |
| 哨兵枚举控制循环 | 显式控制流，避免深层嵌套 |
| 延迟清理 + 恰好一次事件 | 保证事件发射的可靠性 |
| 别名编码引用 | 防止 LLM 篡改资源 ID |

#### 调试技巧

1. **Langfuse Span**：在 Langfuse UI 查看完整的 agent.execute → agent.round.N → chat/tool 嵌套 Span
2. **事件日志**：搜索 `sessionID` 相关的所有事件
3. **工具调用追踪**：每个工具调用前后都有 `EventAgentTool` 和 `EventAgentToolResult` 事件
4. **Token 用量**：`lastUsage` 字段记录最近一次 LLM 调用的 Token 用量

---

### 10.1.2 RAG 管道代码走读

#### 组件概览

RAG 管道采用插件式事件驱动架构，`EventManager` 注册插件并构建职责链。

#### 核心文件关系

```
chat_pipeline/
├── chat_pipeline.go ── EventManager + Plugin 接口 + buildHandler
├── search.go ── 检索插件（混合检索 + 查询扩展）
├── rerank.go ── 重排插件（模型重排 + MMR 多样性）
├── merge.go ── 融合插件（结果合并 + 去重 + 父分块解析）
├── chat_completion.go ── 对话生成插件
├── chat_completion_stream.go ── 流式生成插件
├── into_chat_message.go ── 结果渲染插件
├── query_understand.go ── 查询理解插件
├── load_history.go ── 历史加载插件
├── filter_top_k.go ── Top-K 过滤插件
└── ...
```

#### 插件执行顺序

```
QUERY_UNDERSTAND → SEARCH → RERANK → WEB_FETCH → MERGE →
FILTER_TOP_K → DATA_ANALYSIS → INTO_CHAT_MESSAGE →
LOAD_HISTORY → CHAT_COMPLETION_STREAM
```

#### 扩展方式

```go
// 自定义插件示例
type MyPlugin struct{}

func (p *MyPlugin) ActivationEvents() []string {
    return []string{"MY_CUSTOM_EVENT"}
}

func (p *MyPlugin) OnEvent(ctx context.Context, event *Event) error {
    // 处理事件
    return nil
}

// 注册
eventManager.Register(&MyPlugin{})
```

---

### 10.1.3 IM 适配层代码走读

#### 组件概览

IM 适配层实现 9 个渠道的消息收发，统一 `im.Adapter` 接口。

#### 适配器接口

```go
type Adapter interface {
    Name() string
    SendMessage(ctx context.Context, msg *OutgoingMessage) error
    ParseIncoming(payload []byte) (*IncomingMessage, error)
    VerifySignature(r *http.Request) bool
    HandleWebhook(w http.ResponseWriter, r *http.Request)
}
```

#### 消息生命周期

```
IM 用户发送消息
    ↓
渠道 Webhook/WS 推送到 API Server
    ↓
IMHandler.IMCallback → 验证签名 → 解析消息 → 立即 ACK
    ↓
异步 HandleMessage → 入队 QAQueue
    ↓
Worker 消费 → 分发（文本/图片/音频/命令）
    ↓
RAG/Agent 处理 → 流式生成回复
    ↓
适配器 SendMessage → 长消息分块 → Markdown 渲染 → 发送
```

---

### 10.1.4 MCP 子系统代码走读

#### 组件概览

MCP 子系统实现 Model Context Protocol 客户端，支持 OAuth2 授权的工具调用。

#### 核心流程

```
1. Agent 调用 MCP 工具
2. MCPManager 查找或创建 MCPClient
3. 检查 OAuth Token
   ├── 有效 → 直接调用
   └── 无效/过期 → 触发 OAuth 流程
4. OAuth 流程：
   ├── 生成授权 URL（PKCE + State）
   ├── 发射 EventMCPOAuthRequired
   ├── 用户点击授权 → 外部服务
   ├── 回调 /mcp-oauth/callback
   ├── 交换 Code → Token
   ├── AES-256-GCM 加密存储
   └── 发射 EventMCPOAuthResolved
5. 工具调用 → 返回结果
```

---

### 10.1.5 Wiki 引擎代码走读

#### 组件概览

Wiki 引擎实现从原始文档自动生成结构化互链 Wiki 知识库。

#### 两阶段提取流程

```
Pass 0: WikiCandidateSlugPrompt
├── 输入：文档分块
├── LLM 提取：候选实体/概念骨架
├── 输出：[{name, slug, aliases, description}]
└── 目的：成本优化 + 前缀缓存

Pass 1..N: WikiChunkCitationPrompt
├── 输入：候选 slug + 文档分块
├── LLM 判断：每个 slug 被哪些分块实质性讨论
├── 输出：{sslug: [chunk_id1, chunk_id2, ...]}
└── 目的：建立实体-证据关联
```

#### 并发控制

- **Slug 锁**：`withSlugLock` 确保同一 slug 串行写入
- **Inflight 槽位**：Redis Lua 脚本（`wikiInflightReserveScript`）原子计数
- **Pending-op 队列**：claim/peek/trim + stale-claim recheck

---

### 10.1.6 DI 容器代码走读

#### 组件概览

DI 容器是整个系统的"组装图"，使用 `uber.org/dig` 构造函数注入。

#### 容器构建顺序

```
BuildContainer(c *dig.Container) *dig.Container
├── 1. ResourceCleaner
├── 2. 核心基础设施（Config, Langfuse, DB, File, Redis, Pool）
├── 3. 检索引擎注册表
├── 4. 外部客户端（DocParser, Ollama, Neo4j, Stream, DuckDB）
├── 5. Repository 层（~35 个）
├── 6. MCP 管理器
├── 7. Service 层（~35 个）
├── 8. Web Search 注册表
├── 9. Agent/Session 服务
├── 10. Task Enqueuer（条件分支：asynq 或 sync）
├── 11. Connector Registry
├── 12. DataSource Scheduler
├── 13. Chat Pipeline 插件
├── 14. HTTP Handler（~30 个）
├── 15. IM 适配器（9 个）
├── 16. Router + Asynq Server
└── 17. Wiki 任务恢复
```

#### 条件分支

```go
// Lite vs 分布式
if os.Getenv("REDIS_ADDR") != "" {
    container.Provide(NewAsyncqClient)      // Redis 任务队列
} else {
    container.Provide(NewSyncTaskExecutor)  // 内存 goroutine
}

// 模型并发治理
if redisAvailable {
    container.Provide(NewRedisGovernor)     // Redis 信号量
} else {
    container.Provide(NewLocalGovernor)     // 本地信号量
}
```

---

## 10.2 开发者上手指南

### 10.2.1 环境准备

#### 系统要求

| 组件 | 最低版本 | 推荐版本 |
|------|---------|---------|
| Go | 1.26.0 | 1.26.x |
| Node.js | 20.x | 22.x LTS |
| pnpm | 8.x | 9.x |
| Docker | 24.x | 25.x |
| Docker Compose | 2.x | 2.20+ |
| Python | 3.10 | 3.12 |

#### 克隆项目

```bash
git clone https://github.com/Tencent/WeKnora.git
cd WeKnora
```

### 10.2.2 本地开发环境启动

#### 方式一：Docker Compose（推荐）

```bash
# 1. 复制环境配置
cp .env.example .env

# 2. 编辑 .env（设置 LLM API Key 等）
vim .env

# 3. 启动核心服务
docker compose up -d

# 4. 查看日志
docker compose logs -f app

# 5. 访问
# Web UI: http://localhost
# API: http://localhost:8080
# Swagger: http://localhost:8080/swagger/index.html
```

#### 方式二：快速开发模式

```bash
# 1. 启动基础设施（PostgreSQL + Redis + DocReader）
make dev-start

# 2. 新终端 - 启动后端（Air 热重载）
make dev-app

# 3. 新终端 - 启动前端（Vite 热更新）
make dev-frontend
```

### 10.2.3 代码结构概览

```
WeKnora/
├── cmd/server/main.go          # 入口
├── internal/
│   ├── container/              # DI 容器
│   ├── config/                 # 配置加载
│   ├── router/                 # 路由注册
│   ├── middleware/              # 中间件
│   ├── handler/                # HTTP 处理器
│   ├── application/
│   │   ├── service/            # 业务逻辑
│   │   └── repository/         # 数据访问
│   ├── models/                 # LLM 提供商
│   ├── agent/                  # Agent 引擎
│   ├── im/                     # IM 适配器
│   ├── mcp/                    # MCP 子系统
│   ├── types/                  # 领域类型
│   └── utils/                  # 工具函数
├── frontend/src/               # Vue 3 前端
├── docreader/                  # Python 文档解析
├── migrations/                 # 数据库迁移
└── docs/                       # 文档
```

### 10.2.4 常用开发命令

```bash
# 构建
make build                  # 构建二进制
make build-prod             # 生产构建
make docker-build-all       # 构建所有 Docker 镜像

# 运行
make run                    # 运行
make run-lite               # Lite 模式运行

# 测试
make test                   # 运行测试
go test ./internal/... -v   # 详细测试输出
go test -run TestAgent -v   # 运行特定测试

# 代码质量
make fmt                    # 格式化代码
make lint                   # Lint 检查
make deps                   # 依赖整理

# 数据库
make migrate-up             # 升级数据库
make migrate-down           # 降级数据库
make migrate-create NAME=x  # 创建迁移

# 文档
make docs                   # 生成 Swagger 文档
```

### 10.2.5 调试技巧

#### 后端调试

```bash
# 使用 Delve 调试器
dlv debug cmd/server/main.go

# 或附加到运行中的进程
dlv attach <pid>

# VS Code launch.json 配置
{
    "name": "Debug WeKnora",
    "type": "go",
    "request": "launch",
    "mode": "debug",
    "program": "cmd/server/main.go",
    "env": {
        "GIN_MODE": "debug"
    }
}
```

#### 前端调试

```bash
# 浏览器 DevTools
# Vue Devtools 扩展
# Network 面板查看 API 请求
# Pinia Devtools 查看状态
```

### 10.2.6 提交规范

```
<type>(<scope>): <subject>

type: feat|fix|docs|test|refactor|chore|perf|ci
scope: agent|kb|session|api|ui|config

示例:
feat(agent): add web search tool
fix(kb): handle duplicate file upload
docs(api): update Swagger annotations
```

---

## 10.3 架构决策记录（ADR）

### ADR-001：为什么使用 dig 作为 DI 容器

| 属性 | 值 |
|------|------|
| 状态 | 已接受 |
| 日期 | 2024-01 |
| 决策者 | 核心团队 |

**背景**：需要一个依赖注入容器来管理 ~150 个组件的生命周期。

**选项**：
1. Google Wire（代码生成）
2. Uber dig（反射）
3. 手动构造

**决策**：选择 Uber dig。

**理由**：
- 反射方式无需代码生成步骤，开发体验更好
- `dig.As` 支持接口暴露，`dig.Name` 支持命名 provider
- Go 生态最成熟的 DI 容器

**后果**：
- 编译时无类型检查（运行时 panic 风险）
- 反射有轻微性能开销（启动时一次性，可忽略）

---

### ADR-002：为什么使用 asynq 作为任务队列

| 属性 | 值 |
|------|------|
| 状态 | 已接受 |

**背景**：需要异步任务队列处理文档解析、Wiki 生成等重操作。

**决策**：选择 asynq（Redis 后端）+ SyncTaskExecutor（Lite 模式）。

**理由**：
- asynq 是最流行的 Go 任务队列
- 支持延迟任务、重试、死信队列
- Redis 后端支持分布式
- Lite 模式用内存 goroutine 替代

---

### ADR-003：为什么支持 29 个 LLM Provider

| 属性 | 值 |
|------|------|
| 状态 | 已接受 |

**背景**：企业使用不同的 LLM 提供商，需要避免锁定。

**决策**：统一 `chat.Chat` 接口，支持 29 个 Provider。

**理由**：
- 数据主权：企业可选择本地部署模型
- 成本优化：不同任务用不同模型
- 高可用：Provider 故障时切换

---

### ADR-004：为什么 Wiki 自维护

| 属性 | 值 |
|------|------|
| 状态 | 已接受 |

**背景**：手动维护 Wiki 耗时费力，知识库内容增长后难以管理。

**决策**：Agent 自动生成结构化互链 Wiki + 知识图谱。

**理由**：
- 降低知识维护成本
- 自动保持知识时效性
- 知识图谱提供关联发现能力

---

### ADR-005：为什么 RBAC 4 级

| 属性 | 值 |
|------|------|
| 状态 | 已接受 |

**背景**：需要平衡安全性和易用性。

**决策**：Owner / Admin / Contributor / Viewer 4 级。

**理由**：
- Owner：完全控制
- Admin：管理配置
- Contributor：创建和管理自己的资源
- Viewer：只读访问

---

### ADR-006：为什么 SSE 流式

| 属性 | 值 |
|------|------|
| 状态 | 已接受 |

**背景**：LLM 生成需要较长时间，需要实时反馈。

**决策**：SSE（Server-Sent Events）流式推送。

**理由**：
- 单向推送（服务器 → 客户端），WebSocket 不必要
- 自动重连
- 文本协议，调试友好

---

### ADR-007：为什么 DuckDB 数据分析

| 属性 | 值 |
|------|------|
| 状态 | 已接受 |

**背景**：Agent 需要分析结构化数据（CSV/Excel）。

**决策**：使用 DuckDB 嵌入式 OLAP。

**理由**：
- 零依赖（单二进制）
- SQL 接口
- 列式存储，分析性能优异
- 支持 CSV/Parquet/JSON

---

### ADR-008：为什么 HNSW 索引

| 属性 | 值 |
|------|------|
| 状态 | 已接受 |

**背景**：向量检索需要高效的 ANN 索引。

**决策**：HNSW（Hierarchical Navigable Small World）。

**理由**：
- 召回率高（>95%）
- 查询速度快（对数复杂度）
- pgvector 原生支持
- 支持 1024 维嵌入

---

### ADR-009：为什么 Langfuse 追踪

| 属性 | 值 |
|------|------|
| 状态 | 已接受 |

**背景**：Agent 推理过程需要可观测性。

**决策**：集成 Langfuse 作为唯一追踪后端。

**理由**：
- 开源，可自托管
- 专为 LLM 应用设计
- 支持 Token 用量追踪
- OTLP 标准协议

---

### ADR-010：为什么 Lite 模式

| 属性 | 值 |
|------|------|
| 状态 | 已接受 |

**背景**：开发测试和小规模部署不需要 Redis 等外部依赖。

**决策**：支持 Lite 模式（无 Redis，SQLite + 内存队列）。

**理由**：
- 降低开发环境搭建门槛
- 单文件部署便于测试
- 与分布式模式统一代码库

---

## 10.4 关键算法分析

### 10.4.1 ReAct 循环

**伪代码**：
```
function Execute(query, history):
    state = new AgentState()
    messages = [system_prompt] + history + [query]
    
    for iteration in 1..MaxIterations:
        response = LLM.stream_chat(messages, tools)
        
        if response.has_tool_calls:
            results = parallel_execute(response.tool_calls)
            messages += response + results
            messages = manage_context_window(messages)
        else:
            state.final_answer = response.content
            state.is_complete = true
            break
    
    if not state.is_complete:
        state.final_answer = force_generate_answer(messages)
    
    return state
```

**时间复杂度**：O(MaxIterations × (T_LLM + T_tools))
- T_LLM：LLM 调用时间（主要瓶颈）
- T_tools：工具执行时间

**空间复杂度**：O(MaxIterations × messages)

**优化建议**：
- 并行工具执行（已实现）
- 上下文窗口管理（已实现）
- 缓存 LLM 响应（语义缓存）

---

### 10.4.2 混合检索 + RRF 融合

**伪代码**：
```
function hybrid_search(query, kb_ids):
    # 并行检索
    vector_results = vector_search(query, top_k)  # O(log N)
    keyword_results = bm25_search(query, top_k)   # O(log N)
    
    # RRF 融合
    fused = {}
    for rank, doc in enumerate(vector_results):
        fused[doc.id] += 1.0 / (k + rank + 1)
    for rank, doc in enumerate(keyword_results):
        fused[doc.id] += 1.0 / (k + rank + 1)
    
    # 排序
    return sort_by_score(fused)
```

**时间复杂度**：O(log N + M log M)
- N：向量库大小
- M：融合后结果数

**空间复杂度**：O(M)

**参数**：k=60（RRF 标准值）

---

### 10.4.3 MMR 多样性

**伪代码**：
```
function MMR(results, k, lambda=0.7):
    selected = []
    remaining = results
    
    while len(selected) < k and remaining:
        best = None
        best_score = -inf
        
        for doc in remaining:
            relevance = doc.score
            diversity = max(jaccard(doc, s) for s in selected) if selected else 0
            score = lambda * relevance - (1 - lambda) * diversity
            
            if score > best_score:
                best = doc
                best_score = score
        
        selected.append(best)
        remaining.remove(best)
    
    return selected
```

**时间复杂度**：O(k × n × d)
- k：选择数
- n：候选数
- d：Jaccard 计算维度

**空间复杂度**：O(n)

**参数**：lambda=0.7（相关性权重），k=10

---

### 10.4.4 Token 压缩

**伪代码**：
```
function compress_context(messages, max_tokens):
    system = messages[0]
    current_turn = messages[-1:]
    history = messages[1:-1]
    
    # 分组（assistant + tool_results 为一组）
    groups = group_tool_messages(history)
    
    # 从前面移除直到满足预算
    while estimate_tokens(system + flatten(groups) + current_turn) > max_tokens * 0.8:
        groups.pop_front()
    
    return [system] + flatten(groups) + current_turn
```

**时间复杂度**：O(n × t)
- n：消息数
- t：Token 估算时间

**空间复杂度**：O(n)

**关键约束**：tool_call/tool_result 对永不被拆分

---

### 10.4.5 Wiki 去重

**伪代码**：
```
function dedup_pages(pages):
    candidates = select_candidate_pages(pages)
    
    for page in candidates:
        # LLM 判断是否重复
        prompt = WikiDeduplicationPrompt(pages=[page, *existing])
        result = LLM.generate(prompt)
        
        if result.should_merge:
            merge_into(page, result.target_slug)
        else:
            create_page(page)
    
    return pages
```

**时间复杂度**：O(n × T_LLM)
- n：候选页面数
- T_LLM：LLM 调用时间

**优化建议**：
- 先按名称相似度预过滤（减少 LLM 调用）
- 批量判断（一次 LLM 调用判断多对）

---

### 10.4.6 SSRF 校验

**伪代码**：
```
function validate_url_for_ssrf(url):
    # 1. 解析 URL
    parsed = url_parse(url)
    
    # 2. 协议检查
    if parsed.scheme not in ['http', 'https']:
        return error
    
    # 3. 主机检查
    ips = dns_resolve(parsed.host)
    for ip in ips:
        if is_private_ip(ip) or is_loopback(ip):
            return error
    
    # 4. 重定向链检查（跟随重定向并校验每个跳转）
    redirect_chain = follow_redirects(url)
    for redirect_url in redirect_chain:
        if not validate(redirect_url):
            return error
    
    return ok
```

**时间复杂度**：O(R × D)
- R：重定向链长度
- D：DNS 解析时间

**空间复杂度**：O(R)

---

### 10.4.7 滑动窗口限流

**伪代码**：
```
function rate_limit_check(ip, window=60s, max=10):
    now = time.now()
    key = f"ratelimit:{ip}"
    
    # 移除窗口外的请求
    redis.zremrangebyscore(key, 0, now - window)
    
    # 计算当前窗口内请求数
    count = redis.zcard(key)
    
    if count >= max:
        return reject(429)
    
    # 记录本次请求
    redis.zadd(key, now, now)
    redis.expire(key, window)
    
    return allow
```

**时间复杂度**：O(log N)
- N：窗口内请求数

**空间复杂度**：O(N × IP 数)

---

## 10.5 测试策略

### 10.5.1 测试金字塔

```
        /\
       /  \     E2E 测试（cli-e2e.yml）
      /----\
     /      \   集成测试（httptest）
    /--------\
   /          \ 单元测试（*_test.go, 400+ 文件）
  /------------\
```

### 10.5.2 单元测试

**覆盖范围**：
- 核心业务逻辑（Agent 引擎、RAG 管道、Wiki 引擎）
- 中间件（认证、RBAC、限流）
- 工具函数（加密、SSRF 校验、Token 估算）

**测试框架**：
- `testing`（标准库）
- `testify/assert` + `testify/mock`
- `sqlmock`（数据库 mock）
- `miniredis`（Redis mock）
- `httptest`（HTTP mock）

**表驱动测试示例**：
```go
func TestKnowledgeSearch(t *testing.T) {
    tests := []struct {
        name     string
        input    KnowledgeSearchInput
        expected *ToolResult
        wantErr  bool
    }{
        {
            name: "basic search",
            input: KnowledgeSearchInput{
                Queries:         []string{"test"},
                KnowledgeBaseIDs: []string{"kb-1"},
            },
            expected: &ToolResult{Success: true},
        },
        {
            name: "empty query",
            input: KnowledgeSearchInput{
                Queries: []string{},
            },
            wantErr: true,
        },
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            tool := NewKnowledgeSearchTool(...)
            result, err := tool.Execute(ctx, marshal(tt.input))
            if tt.wantErr {
                assert.Error(t, err)
            } else {
                assert.NoError(t, err)
                assert.Equal(t, tt.expected.Success, result.Success)
            }
        })
    }
}
```

### 10.5.3 集成测试

**覆盖范围**：
- API 端到端测试
- 数据库集成测试（testcontainers）
- LLM Provider 集成测试（mock server）

**关键测试文件**：
| 文件 | 测试内容 |
|------|---------|
| `agent/engine_test.go` | Agent 引擎执行 |
| `agent/engine_image_test.go` | Agent 图片处理 |
| `handler/embed_flow_test.go` | 嵌入端到端流程 |
| `handler/datasource_test.go` | 数据源同步流程 |
| `service/wiki_ingest_test.go` | Wiki 摄入全流程 |

### 10.5.4 E2E 测试

**CLI E2E**（`.github/workflows/cli-e2e.yml`）：
1. 启动完整 Docker Compose 环境
2. 执行 CLI 命令（login/kb list/doc upload/chat）
3. 验证输出

**前端 E2E**（建议补充）：
- Playwright / Cypress
- 覆盖核心用户旅程（登录→创建 KB→上传文档→问答）

### 10.5.5 性能测试

**建议基准测试**：
```go
func BenchmarkKnowledgeSearch(b *testing.B) {
    tool := setupKnowledgeSearchTool()
    for i := 0; i < b.N; i++ {
        tool.Execute(ctx, benchmarkQuery)
    }
}
```

**关键指标**：
| 指标 | 目标 | 测试方法 |
|------|------|---------|
| 知识检索 P95 | < 200ms | 并发 100 请求 |
| RAG 问答 P95 | < 3s | 并发 50 请求 |
| Agent 首 Token | < 2s | 并发 20 请求 |
| 文档解析 | < 30s/10MB | 单文件顺序处理 |
| 并发用户 | 100+ | 逐步加压 |

### 10.5.6 安全测试

| 测试类型 | 工具 | 频率 |
|---------|------|------|
| 依赖漏洞扫描 | Trivy / Snyk | 每次构建 |
| 静态代码分析 | GolangCI-Lint | 每次提交 |
| SAST | Semgrep | 每周 |
| DAST | OWASP ZAP | 每月 |
| 渗透测试 | 人工 | 每季度 |

### 10.5.7 测试数据管理

**测试数据位置**：
- `testdata/`：测试用 JSON/文件
- `*_test.go` 内联 fixtures
- `migrations/`：迁移测试

**Mock 策略**：
| 依赖 | Mock 方式 |
|------|----------|
| 数据库 | sqlmock / SQLite 内存 |
| Redis | miniredis |
| LLM Provider | httptest mock server |
| 对象存储 | DummyFileService |
| DocReader | gRPC mock |

---

## 10.6 总结

本章提供了 WeKnora 的完整增强文档：

1. **代码走读文档**：6 个核心组件的详细执行流程、关键设计决策、调试技巧
2. **开发者上手指南**：环境准备、启动流程、调试技巧、提交规范
3. **架构决策记录**：10 个重要决策的背景、选项、理由、后果
4. **关键算法分析**：7 个核心算法的伪代码、复杂度分析、优化建议
5. **测试策略**：测试金字塔、单元/集成/E2E/性能/安全测试方案

---

## 文档完结

至此，WeKnora v0.7.1 完整设计文档与代码讲解已全部输出完毕。文档包含：

| 章节 | 文件 | 内容 |
|------|------|------|
| 第 1 章 | [01-project-overview.md](./01-project-overview.md) | 项目概述 |
| 第 2 章 | [02-c4-architecture.md](./02-c4-architecture.md) | C4 架构模型 |
| 第 3 章 | [03-flows-sequence.md](./03-flows-sequence.md) | 系统流程与时序图 |
| 第 4 章 | [04-module-structure.md](./04-module-structure.md) | 模块结构与依赖分析 |
| 第 5 章 | [05-code-walkthrough.md](./05-code-walkthrough.md) | 核心代码深度走读 |
| 第 6 章 | [06-data-model.md](./06-data-model.md) | 数据模型与数据库设计 |
| 第 7 章 | [07-api-design.md](./07-api-design.md) | API 与接口设计 |
| 第 8 章 | [08-deploy-ops.md](./08-deploy-ops.md) | 部署运维与基础设施 |
| 第 9 章 | [09-improvements.md](./09-improvements.md) | 改进建议与风险点 |
| 第 10 章 | [10-enhanced-content.md](./10-enhanced-content.md) | 增强内容 |

**总输出**：10 个 Markdown 文件，位于 `docs/wangbin/` 目录，涵盖 238 个 API 端点、1,492 个 Go 文件、37.5 万行代码的完整分析。

---

☕️ 制作不易，请我喝咖啡☕️关注我➕