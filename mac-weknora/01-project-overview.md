# 第 1 章 项目概述（Project Overview）

> **WeKnora v0.7.1** — 腾讯开源的 LLM 驱动企业知识框架，以 RAG 快速问答、ReAct 智能体、Wiki 自动知识库三大核心能力，将散落的文档转化为可查询、可推理、持续演进的知识资产。

---

## 1.1 项目背景与目标

### 1.1.1 核心价值主张

WeKnora（"We Know + Nora[知识]"）旨在解决企业知识管理中的三大痛点：

1. **知识碎片化**：文档散落在飞书、Notion、语雀、RSS、本地文件等多个孤岛，缺乏统一的语义检索入口。
2. **问答效率低**：传统关键词匹配无法理解语义，员工需要翻阅大量文档才能找到答案。
3. **知识不生长**：静态文档无法自动关联、自动演进，知识库维护成本极高。

WeKnora 通过以下方式实现价值闭环：

- **RAG 快速问答**（Quick Q&A）：基于向量 + 关键词 + 图谱的混合检索，秒级返回带引用的精准答案。
- **ReAct 智能体**（Agent Mode）：多步推理自主编排知识检索、MCP 工具调用、网络搜索，处理复杂任务。
- **Wiki 模式**（Wiki Mode）：Agent 自动从原始文档蒸馏出结构化、互链的 Markdown Wiki 知识体系，并生成知识图谱。
- **多源数据接入**：自动同步飞书、Notion、语雀、RSS 等外部数据源，支持 10+ 文档格式。
- **全渠道触达**：通过 Web UI、CLI、Chrome 扩展、微信小程序、网站嵌入控件、企业微信/飞书/Slack/Telegram/钉钉/QQ 机器人等 9 个 IM 渠道提供服务。

### 1.1.2 解决的问题

| 问题维度 | 传统方案痛点 | WeKnora 解决方案 |
|---------|------------|----------------|
| 检索精度 | 关键词匹配召回率低 | BM25 + 向量 + GraphRAG 混合检索 + Rerank 重排 |
| 多步推理 | 单次检索无法回答复杂问题 | ReAct Agent 自主编排多轮工具调用 |
| 知识维护 | 手动整理 Wiki 耗时费力 | Agent 自动生成互链 Wiki 页面 + 知识图谱 |
| 数据孤岛 | 多平台文档无法统一检索 | 多源连接器自动同步 + 统一语义索引 |
| 部署灵活性 | SaaS 方案数据不出域 | 全模块化自托管，支持本地/私有云/离线部署 |
| 模型锁定 | 绑定单一 LLM 提供商 | 29 个 LLM Provider、12+ 向量库可热插拔 |
| 企业协作 | 缺乏权限隔离 | 4 级 RBAC + 多工作空间 + 资源级所有权 |
| 可观测性 | 黑盒推理过程 | Langfuse 全链路追踪 + Token 用量监控 |

### 1.1.3 目标用户画像

- **企业知识团队**：需要统一管理内部文档、FAQ、Wiki 的知识管理部门
- **技术团队 / DevOps**：需要快速检索技术文档、Runbook、架构决策记录的工程师
- **客户支持团队**：需要基于知识库快速回答客户问题的客服/技术支持
- **AI 开发者**：需要 RAG + Agent 框架构建知识密集型应用的开发者
- **个人用户**：需要管理个人笔记、论文、知识体系的知识工作者

---

## 1.2 技术栈完整清单

### 1.2.1 服务端技术栈

| 类别 | 技术选型 | 版本/说明 |
|------|---------|----------|
| 编程语言 | **Go** | 1.26.0 |
| Web 框架 | **Gin** | v1.12.0（HTTP 路由 + 中间件） |
| 依赖注入 | **dig** (uber-go) | 构造函数 DI，全容器 ~150 个 provider |
| ORM | **GORM** | 支持 PostgreSQL / MySQL / SQLite |
| 数据库迁移 | **golang-migrate** | v4.19.1，多引擎 versioned 迁移 |
| 任务队列 | **asynq** (Redis) / **SyncTaskExecutor** | Lite 模式用内存 goroutine，分布式用 Redis |
| 缓存/流 | **go-redis** | Redis 流管理 + TLS 支持 |
| 认证 | **golang-jwt** (JWT v5) | Bearer Token + Refresh Token |
| 配置 | **Viper** + **mapstructure** | YAML + 环境变量覆盖 |
| 验证 | **go-openapi/strfmt** + **google/jsonschema-go** | 参数 JSON Schema 校验 |
| 日志 | 自研 logger | 结构化日志 + 敏感信息脱敏 |
| 可观测性 | **Langfuse** (OTLP/OTel) | 全链路追踪 + W3C traceparent 传播 |
| 并发池 | **ants** | 高性能 goroutine 池 |
| 向量运算 | **sqlite-vec**、**pgvector** | HNSW 索引，1024 维嵌入 |
| 文档解析 | gRPC/HTTP 调用 docreader | MinerU、PaddleOCR-VL、内置转换器 |
| 数据分析 | **DuckDB** (go) | 嵌入式 OLAP，Agent 数据分析工具 |
| 安全 | AES-256-GCM、SSRF-safe HTTP Client | 密钥加密 + 出站 URL 校验 |
| 文档生成 | **swaggo/gin-swagger** | OpenAPI/Swagger 自动文档 |
| LLM SDK | 自研多 Provider 适配 | 29 个 Provider，统一 Chat/Embedding/Rerank/VLM/ASR 接口 |

### 1.2.2 前端技术栈

| 类别 | 技术选型 | 版本/说明 |
|------|---------|----------|
| 框架 | **Vue 3** | v3.5.34，Composition API |
| 语言 | **TypeScript** | ~6.0 |
| 构建工具 | **Vite** | v7.3.5 |
| 状态管理 | **Pinia** | v3.0.4 |
| UI 组件库 | **TDesign Vue Next** | v1.19.2 |
| 路由 | **Vue Router** | v4.5.0 |
| 国际化 | **Vue I18n** | v11.4.2 |
| HTTP 客户端 | **Axios** | v1.16.0 |
| SSE 流 | **@microsoft/fetch-event-source** | v2.0.1 |
| Markdown 渲染 | **marked** + **highlight.js** | v17 + v11 |
| 数学公式 | **KaTeX** | v0.16 |
| 图表 | **Mermaid** | v11 |
| 文档预览 | **docx-preview**、**@vue-office/pptx** | Word/PPT 在线预览 |
| 安全 | **DOMPurify** | XSS 防护 |
| 虚拟滚动 | **vue-virtual-scroller** | 大数据列表 |
| 全文检索 | **pagefind** | 静态搜索索引 |

### 1.2.3 其他组件

| 组件 | 技术栈 | 说明 |
|------|--------|------|
| **docreader** (文档解析服务) | Python + gRPC/HTTP | 52 个 Python 文件，MinerU/PaddleOCR-VL/内置转换器 |
| **mcp-server** | Python | 10 个文件，MCP 协议 stdio/SSE/HTTP 多传输 |
| **CLI (weknora)** | Go | 284 个 Go 文件，Agent-first JSON 信封 |
| **client SDK** | Go | 28 个文件，官方 Go SDK |
| **Desktop** | Wails (Go + Vue) | 桌面客户端 |
| **微信小程序** | 原生小程序 | 轻量移动端 |
| **Chrome Extension** | 浏览器扩展 | 网页内容一键入库 |

### 1.2.4 基础设施与运维

| 类别 | 技术选型 | 说明 |
|------|---------|------|
| 容器化 | Docker + Docker Compose | 多服务编排（docker-compose.yml 30+ 服务） |
| 编排 | Helm Chart | Kubernetes 部署（Chart.yaml + templates） |
| CI/CD | GitHub Actions | cli-e2e / cli / docker-image / release-lite 4 个 workflow |
| 数据库 | PostgreSQL / MySQL / SQLite | 关系数据 |
| 向量数据库 | pgvector / Milvus / Qdrant / Weaviate / Elasticsearch / OpenSearch / Doris / Tencent VectorDB / Neo4j | 12+ 向量/图谱后端 |
| 对象存储 | Local / MinIO / AWS S3 / Volcengine TOS / Alibaba OSS / KS3 / OBS | 多实例存储后端 |
| 缓存/队列 | Redis | 流管理 + 任务队列 + 限流 + 发布订阅 |
| 可观测性 | Langfuse | 全链路追踪、Token 用量、Agent 推理过程 |

---

## 1.3 架构风格与设计哲学

### 1.3.1 分层架构（Layered Architecture）

WeKnora 采用经典的分层架构，自顶向下为：

```
┌──────────────────────────────────────────────────┐
│  接入层 (Access Layer)                            │
│  Web UI / CLI / API / IM / Embed / MiniProgram   │
├──────────────────────────────────────────────────┤
│  路由层 (Router + Middleware)                     │
│  Gin Engine → Auth → RBAC → APIKeyGate → Handler │
├──────────────────────────────────────────────────┤
│  应用层 (Application / Service)                   │
│  AgentService / KnowledgeService / SessionService │
├──────────────────────────────────────────────────┤
│  领域层 (Domain / Types / Interfaces)             │
│  Entity + Interface contracts                     │
├──────────────────────────────────────────────────┤
│  基础设施层 (Infrastructure / Repository)          │
│  DB / Vector Store / LLM / Storage / Cache        │
└──────────────────────────────────────────────────┘
```

### 1.3.2 依赖注入（Dependency Injection）

全量使用 `uber.org/dig` 构造函数注入，`internal/container/container.go`（1674 行）是唯一的依赖注册中心，负责将 ~150 个 provider（DB、Redis、Repository、Service、Handler、Router、IM Adapter、Task Server）组装为完整对象图。

**设计理由**：
- 解耦组件依赖，便于单元测试时替换 mock
- 单一注册点，启动顺序清晰
- 支持 Lite 模式（无 Redis）与分布式模式的条件编译分支

### 1.3.3 接口隔离（Interface Segregation）

`internal/types/interfaces/` 目录下定义了 **48 个接口**（如 `KnowledgeService`、`KnowledgeBaseService`、`SessionService`、`MessageService`、`TaskEnqueuer`、`RetrieveEngineService` 等），每个接口职责单一。

**设计理由**：
- 上层依赖抽象，底层实现可替换
- 便于 mock 测试
- 支持多实现（如 `TaskEnqueuer` 有 `asynq.Client` 和 `SyncTaskExecutor` 两个实现）

### 1.3.4 事件驱动（Event-Driven）

`internal/event/` 包实现进程内 EventBus，Agent 推理过程中的每个步骤（思考、工具调用、工具结果、最终答案）都通过事件发射，前端通过 SSE 流实时消费。

**设计理由**：
- 解耦 Agent 执行与结果渲染
- 支持多消费者（SSE、日志、审计）
- 事件持久化后可回放

### 1.3.5 策略模式（Strategy）

- **检索策略**：BM25 / 向量 / GraphRAG / parent-child chunking 可配置
- **分块策略**：heading / heuristic / adaptive 3-tier chunking
- **LLM Provider**：29 个 Provider 统一 `chat.Chat` 接口
- **向量库**：10 个引擎统一 `RetrieveEngineService` 接口
- **IM 适配器**：9 个渠道统一 `im.Adapter` 接口
- **部署模式**：Lite（内存）vs 分布式（Redis/asynq）

### 1.3.6 管道模式（Pipeline）

RAG 问答采用可组合的 `chat_pipeline` 插件链：

```
Query Understand → Search → Rerank → Web Fetch → Merge →
Filter Top-K → Data Analysis → Into Chat Message → Chat Completion Stream
```

每个阶段是可插拔的 `PipelinePlugin`，可按需启用/重排。

---

## 1.4 关键功能特性矩阵

### 1.4.1 智能对话（Intelligent Conversation）

| 功能 | 描述 | 核心模块 |
|------|------|---------|
| ReAct 推理 | 多步渐进式推理，自主编排知识检索、MCP 工具、网络搜索 | `internal/agent/` |
| 快速 Q&A | 基于知识库的 RAG 问答 | `chat_pipeline/` |
| Wiki 模式 | Agent 自动生成结构化互链 Markdown Wiki | `prompts_wiki.go`、`wiki_ingest/` |
| 工具调用 | 内置工具 + MCP 工具（含 OAuth2）+ 网络搜索 | `agent/tools/` |
| 会话策略 | Prompt 在线编辑、检索阈值调优、多轮上下文感知 | `ConversationConfig` |
| 问题建议 | 基于知识库内容的自动问题推荐 | `message_suggestion.go` |
| 临时附件 | 会话级图片/文档上传，异步解析 | `temporary_document.go` |
| 引用与进度 | 内联引用气泡 + 引用抽屉 + RAG 进度条 | 前端 `useChatCitationPopover` |
| 会话管理 | 按来源（Web/IM/Embed）筛选分组 | `session/` handler |

### 1.4.2 知识管理（Knowledge Management）

| 功能 | 描述 | 核心模块 |
|------|------|---------|
| 知识库类型 | FAQ / Document / Wiki，支持文件夹导入、URL 导入 | `knowledgebase.go` |
| 按上传配置 | 每次上传可覆盖解析器/分块/多模态/图谱/问答生成配置 | `knowledge_process_config.go` |
| 批量重解析 | 多文档批量重新解析 | `knowledge.go` handler |
| 数据源同步 | 飞书/Notion/语雀/RSS 自动同步（增量+全量） | `datasource/connector/` |
| 文档格式 | PDF/Word/Txt/Markdown/HTML/EPUB/MHTML/图片/CSV/Excel/PPT/JSON | `infrastructure/docparser/` |
| 检索策略 | BM25/向量/GraphRAG/parent-child/HNSW pgvector/多维索引 | `knowledgebase_search_*` |
| 批量选择 | 拖拽框选多文档 | 前端 `useMarqueeSelect` |
| E2E 评测 | 召回率、BLEU/ROUGE 指标可视化 | `evaluation.go` |

### 1.4.3 集成与扩展（Integrations & Extensions）

| 类别 | 支持列表 |
|------|---------|
| LLM | OpenAI / Azure OpenAI / Anthropic / DeepSeek / 通义千问 / 智谱 / 混元 / 豆包 / Gemini / MiniMax / NVIDIA / Novita AI / SiliconFlow / OpenRouter / Requesty / Ollama 等 29 个 |
| Embedding | Ollama / BGE / GTE / Zhipu / OpenAI 兼容 |
| 向量库 | pgvector / Elasticsearch / OpenSearch / Milvus / Weaviate / Qdrant / Doris / Tencent VectorDB 等 12+ |
| 对象存储 | Local / MinIO / AWS S3 / Volcengine TOS / Alibaba OSS / KS3 / OBS，支持多实例 |
| IM 渠道 | 企业微信 / 飞书 / 国际飞书 / QQ Bot / Slack / Telegram / 钉钉 / Mattermost / 微信 / 云之家 |
| 网站嵌入 | 域名白名单 + 限流 + 安全模式 Token 交换 |
| 网络搜索 | DuckDuckGo / Bing / Google / Tavily / Baidu / Ollama / SearXNG / Keenable / 智谱 |
| API 集成 | 作用域 API Key（能力级授权 + 知识库限制 + 限流）|

### 1.4.4 平台能力（Platform）

| 功能 | 描述 |
|------|------|
| 部署 | 本地 / Docker / Kubernetes (Helm)，支持私有/离线 |
| 界面 | Web UI / RESTful API / CLI / Chrome 扩展 / 嵌入控件 / 小程序 |
| 访问控制 | 4 级 RBAC（Owner/Admin/Contributor/Viewer）+ 资源所有权 + 审计日志 |
| 安全 | AES-256-GCM 加密、gRPC TLS、Redis TLS、SSRF 安全、沙箱隔离 |
| 可观测性 | Langfuse 全链路追踪 + Token 用量 + 任务队列仪表盘 |
| 任务管理 | MQ 异步任务 + 分阶段 Worker-Pool + 按模型并发治理 |
| 模型管理 | 集中配置 + YAML 声明 + 按知识库选模型 + Thinking 模式 + 测试调试器 |

---

## 1.5 非功能性需求

### 1.5.1 性能（Performance）

| 指标 | 目标/策略 |
|------|----------|
| 问答延迟 | RAG 问答 P95 < 3s（不含 LLM 生成），Agent 首 Token < 2s |
| 检索速度 | HNSW 索引支持 1024 维嵌入的毫秒级 ANN 查询 |
| 流式输出 | SSE 流式推送，首字节时间（TTFB）优化 |
| 并发处理 | ants goroutine 池管理并发，Redis 全局 Worker 上限 |
| 嵌入批处理 | `embedding/batch.go` 批量嵌入减少 API 调用 |
| 并发包装 | `concurrency_wrapper.go` 按模型并发控制 |
| 提示缓存 | `prompt_cache.go` 利用 Provider Prompt Cache 降本 |
| 上下文管理 | Token 估算 + LLM 压缩 + 确定性裁剪，防止窗口溢出 |

### 1.5.2 扩展性（Scalability）

| 维度 | 策略 |
|------|------|
| 水平扩展 | 无状态 API 实例 + Redis 共享会话/队列，支持多副本 |
| 存储扩展 | 多存储后端实例，按知识库绑定，支持热添加 |
| 向量库扩展 | 12+ 向量引擎，运行时通过 API 创建，无需重启 |
| LLM 扩展 | 29 个 Provider，统一接口，新增 Provider 只需实现 5 个方法 |
| IM 扩展 | 9 个渠道适配器，统一 `Adapter` 接口 |
| 任务扩展 | 6 个独立 Worker-Pool（core/postprocess/enrichment/maintenance/shared/wiki），容量隔离 |
| 插件扩展 | `chat_pipeline` 可组合插件链 |

### 1.5.3 安全性（Security）

| 层面 | 措施 |
|------|------|
| 认证 | JWT Bearer + Refresh Token、API Key（Tenant/Platform）、Embed Session、OIDC、微信扫码 |
| 授权 | 4 级 RBAC + 资源所有权 + KB 访问控制（自有/组织共享/Agent 共享）|
| 加密 | AES-256-GCM 静态加密（API Key、MCP 凭证、数据源凭证）、gRPC TLS、Redis TLS |
| 网络安全 | SSRF-safe HTTP Client（数据源、URL 导入、重定向链）、CORS、Trusted Proxies |
| 输入安全 | JSON Schema 校验、参数消毒、DOMPurify XSS 防护 |
| 沙箱 | Agent Skills 支持 Docker/Local 沙箱隔离执行 |
| 审计 | 操作审计日志 + 知识库活动审计 + 系统审计（三级审计）|
| 密钥轮换 | AES 密钥平滑轮换，不影响已加密数据 |
| 敏感信息 | 日志脱敏（密码/Token/API Key/Secret）、响应中密钥掩码 |

### 1.5.4 可用性（Availability）

| 策略 | 说明 |
|------|------|
| 优雅关闭 | 信号监听 → Listener 关闭 → 连接 Drain → 清理回调（LIFO 顺序）|
| 健康检查 | `/health` 端点 |
| 任务恢复 | 启动时重置卡住的知识/摘要/同步任务（`resetPendingTasks`）|
| Wiki 任务恢复 | 重启后重新触发待处理的 Wiki 队列（`recoverPendingWikiTasks`）|
| 死信队列 | asynq DLQ + 失败回调标记知识为失败 |
| 重试机制 | 指数退避 + 最大重试次数 + 瞬态错误检测 |
| 故障降级 | LLM 调用失败时优雅降级为已有工具结果生成答案 |

### 1.5.5 可维护性（Maintainability）

| 实践 | 说明 |
|------|------|
| 代码规范 | `gofmt` + `.golangci.yml` lint + Conventional Commits |
| 测试覆盖 | 单元测试 + 集成测试 + E2E（`*_test.go` 文件 400+）|
| 文档 | Swagger/OpenAPI 自动文档 + 多语言 README（中/英/日/韩）|
| 模块化 | 每个组件可独立替换（LLM/向量库/存储/IM）|
| 配置即代码 | YAML 声明式配置 + 环境变量覆盖 |
| 可观测性 | Langfuse 追踪 + 结构化日志 + 运行时任务仪表盘 |

### 1.5.6 数据主权（Data Sovereignty）

WeKnora 支持完全自托管部署，所有数据（文档、向量、元数据、缓存）均存储在用户指定的基础设施中，支持：
- 离线部署（无外网访问）
- 私有云部署
- 本地 SQLite 单文件部署（Lite 模式）
- 数据不出域（符合企业合规要求）

---

## 1.6 项目规模统计

| 指标 | 数值 |
|------|------|
| Go 源码文件 | 1,492 个 |
| Go 代码行数 | ~375,000 行 |
| 前端文件 | 464 个 |
| 前端代码行数 | ~182,000 行 |
| Python 文件 (docreader+mcp) | ~62 个 |
| CLI Go 文件 | 284 个 |
| 数据库迁移 | 4 个引擎 |
| LLM Provider | 29 个 |
| 向量数据库后端 | 12+ |
| IM 渠道适配器 | 9 个 |
| HTTP 端点 | ~150+ |
| 接口定义 | 48 个 |
| 环境变量 | ~200+（`.env.example` 37KB）|

---

## 1.7 版本演进里程碑

| 版本 | 关键特性 |
|------|---------|
| v0.2.0 | Agent Mode (ReACT)、多类型知识库、MCP 工具集成、MQ 异步任务 |
| v0.3.0 | 共享空间、Agent Skills、自定义 Agent、思维模式、Web 搜索、Helm |
| v0.3.4 | 企业微信/飞书/Slack IM、多模态图片、AES-256-GCM 加密、混合搜索优化 |
| v0.4.0 | WeKnora Cloud、Chrome 扩展、微信小程序、Notion 连接器 |
| v0.5.0 | Wiki Mode GA — Agent 自动生成互链 Wiki + 知识图谱 |
| v0.6.0 | 工作空间 RBAC、4 级角色矩阵、Go 1.26 |
| v0.7.0 | 作用域 API Key、运行时任务队列仪表盘、多实例存储后端、会话临时附件 |
| **v0.7.1** | 云之家 IM、火山引擎 Rerank、智谱网络搜索、平台级 API Key、Langfuse OTLP 迁移 |

---

> **下一章**：[第 2 章 C4 架构模型](./02-c4-architecture.md) — 从 Context、Container、Component 到 Code 四层完整剖析系统架构。

---

☕️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../res/promotion.jpg)