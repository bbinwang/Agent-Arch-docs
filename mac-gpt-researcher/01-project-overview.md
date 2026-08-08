# 第1章 项目概述 (Project Overview)

> **文件**: `docs/wangbin/01-project-overview.md`  
> **预计 Token**: ~12,000  
> **核心内容**: 项目目标、价值、技术栈、架构风格、非功能需求

---

## 1.1 项目定位与目标

### 1.1.1 项目定义

**GPT Researcher** 是首个开源的**深度研究智能体 (Deep Research Agent)**，专为任意研究任务的 Web 和本地调研而设计。它能够自主生成**详细、真实、无偏见且带引用来源的研究报告**。

项目由 **Assaf Elovic** 创建并维护，当前版本 **v0.14.7**，采用 **MIT 许可证**开源。官方网站为 [gptr.dev](https://gptr.dev)，文档站为 [docs.gptr.dev](https://docs.gptr.dev)。

### 1.1.2 核心价值主张

GPT Researcher 解决以下核心痛点：

| 痛点 | 传统方式 | GPT Researcher 方案 |
|------|---------|-------------------|
| **研究周期长** | 人工调研需数周 | 数分钟完成深度报告 |
| **LLM 幻觉** | 训练数据过时导致错误 | 实时联网检索获取最新信息 |
| **Token 限制** | 单窗口无法容纳长报告 | 分块聚合 + 上下文压缩 |
| **信息源单一** | 有限来源导致片面结论 | 聚合 20+ 来源交叉验证 |
| **来源偏见** | 选择性引用引入主观偏向 | 多源比较 + 来源审查 |

### 1.1.3 设计灵感

项目核心设计受以下学术论文启发：

1. **Plan-and-Solve** (arXiv:2305.04091) — 规划-执行分离范式
   - 将研究任务分解为规划阶段和执行阶段
   - Planner 生成研究问题，Executor 并行收集信息
   
2. **RAG (Retrieval-Augmented Generation)** (arXiv:2005.11401) — 检索增强生成
   - 将生成模型与检索系统结合
   - 通过外部知识源增强 LLM 的事实准确性

3. **深度研究 (Deep Research)** — 递归式探索
   - 基于初始发现生成后续问题
   - 多层递归以达成深度理解

### 1.1.4 目标用户群体

| 用户类型 | 使用场景 | 核心需求 |
|---------|---------|---------|
| **个人研究者** | 快速了解新领域 | 速度、准确性、引用 |
| **企业分析师** | 市场竞争分析 | 全面性、多源验证 |
| **学术工作者** | 文献综述辅助 | 学术来源、无偏见 |
| **新闻从业者** | 事实核查、深度报道 | 实时性、来源追踪 |
| **开发者** | 构建研究类应用 | API、可定制性 |
| **产品经理** | 技术调研、竞品分析 | 结构化输出、可读性 |

---

## 1.2 项目目标与使命

### 1.2.1 使命宣言

> *"Our mission is to empower individuals and organizations with accurate, unbiased, and factual information through AI."*
> 
> "我们的使命是通过 AI 为个人和组织提供准确、无偏见、事实性的信息。"

### 1.2.2 核心目标

1. **客观性 (Objectivity)**: 通过多源交叉验证消除单一来源偏见
2. **准确性 (Accuracy)**: 实时联网检索确保信息时效性
3. **深度 (Depth)**: 递归探索机制实现超越浅层搜索的深度理解
4. **可追溯 (Traceability)**: 每条结论附带引用来源
5. **可定制 (Customizability)**: 丰富的配置选项适应不同领域需求
6. **可扩展 (Extensibility)**: 插件化架构支持新检索器/爬虫/LLM 接入

### 1.2.3 关键成果指标

| 指标 | 目标值 |
|------|-------|
| 单报告来源数 | 20+ |
| 报告长度 | 2,000+ 字 |
| 标准报告生成时间 | ~2 分钟 |
| 深度报告生成时间 | ~5–15 分钟 |
| 支持的语言模型 | 27 家 |
| 支持的搜索引擎 | 20+ 家 |

---

## 1.3 技术栈完整清单

### 1.3.1 编程语言与运行时

| 类别 | 技术 | 版本 | 说明 |
|------|------|------|------|
| 后端语言 | Python | >= 3.11 | 核心 Agent 逻辑 |
| 前端语言 | TypeScript/React | NextJS 14 | 生产级前端 |
| 前端语言 | 原生 HTML/CSS/JS | - | 轻量前端 |
| 运行时 | Node.js | - | 前端构建 |
| 运行时 | Uvicorn | >= 0.34.2 | ASGI 服务器 |

### 1.3.2 核心框架

| 框架 | 版本 | 用途 |
|------|------|------|
| **FastAPI** | >= 0.104.1 | Web API 框架 |
| **LangChain** | >= 1.0.0 | LLM 编排框架 |
| **LangChain Core** | >= 1.0.0 | 核心抽象 |
| **LangChain Community** | >= 0.4.0 | 社区集成 |
| **LangChain OpenAI** | >= 1.0.0 | OpenAI 集成 |
| **LangGraph** | >= 0.2.73, < 0.3 | 多 Agent 状态机 |
| **Pydantic** | >= 2.11.5 | 数据验证 |
| **Starlette** | >= 0.46.2 | ASGI 框架 (FastAPI 底层) |
| **SQLAlchemy** | >= 2.0.41 | ORM (向量存储底层) |

### 1.3.3 LLM 提供商支持 (27 家)

| 提供商 | 集成包 | 说明 |
|--------|-------|------|
| OpenAI | langchain_openai | GPT-5.4/5/4.1/4o, O3/O4 |
| Anthropic | langchain_anthropic | Claude 4.x/4.5/4.6/4.7 |
| Azure OpenAI | langchain_openai | 企业级 OpenAI |
| Google VertexAI | langchain_google_vertexai | GCP 托管 |
| Google GenAI | langchain_google_genai | Gemini 系列 |
| Ollama | langchain_ollama | 本地模型 |
| Together | langchain_together | 开源模型托管 |
| MistralAI | langchain_mistralai | Mistral 系列 |
| Cohere | langchain_cohere | Command 系列 |
| Fireworks | langchain_fireworks | 高速推理 |
| Groq | langchain_groq | LPU 推理 |
| HuggingFace | langchain_huggingface | 开源模型 |
| Bedrock | langchain_aws | AWS 托管 |
| DashScope | langchain_community | 通义千问 |
| xAI | - | Grok 系列 |
| DeepSeek | - | DeepSeek 系列 |
| LiteLLM | litellm | 统一代理 |
| GigaChat | langchain_gigachat | 俄罗斯 Sber |
| OpenRouter | - | 多模型路由 |
| vLLM (OpenAI) | - | 本地高吞吐 |
| AIMLAPI | - | 聚合 API |
| Netmind | langchain_netmind | GPU 云 |
| Forge | - | 定制推理 |
| Avian | - | 新兴提供商 |
| MiniMax | - | 国产大模型 |
| AtlasCloud | - | 云推理 |
| Nebius | - | Token Factory |

### 1.3.4 检索后端 (20+)

| 检索器 | 类型 | 说明 |
|--------|------|------|
| **Tavily** | Web 搜索 | 默认推荐，AI 优化 |
| **DuckDuckGo** | Web 搜索 | 免费，隐私友好 |
| **Google** | Web 搜索 | Custom Search API |
| **Bing** | Web 搜索 | Microsoft Search API |
| **Brave** | Web 搜索 | 独立搜索引擎 |
| **Serper** | Web 搜索 | Google 代理 |
| **SerpAPI** | Web 搜索 | 多引擎代理 |
| **SearchAPI** | Web 搜索 | 结构化搜索 |
| **SearX** | Web 搜索 | 自托管元搜索 |
| **Exa** | Web 搜索 | 语义搜索 |
| **GroundRoute** | Web 搜索 | 路由搜索 |
| **CRW** | Web 搜索 | Firecrawl 兼容 |
| **Bocha** | Web 搜索 | 中文搜索 |
| **ArXiv** | 学术搜索 | 物理学/CS 论文 |
| **Semantic Scholar** | 学术搜索 | 微软学术 |
| **PubMed Central** | 医学文献 | NIH 医学文献 |
| **OpenAlex** | 学术目录 | 开放学术图谱 |
| **Xquik** | 社交搜索 | X/Twitter 搜索 |
| **GetXAPI** | 社交搜索 | X/Twitter API |
| **MCP** | 工具协议 | Model Context Protocol |
| **Custom** | 自定义 | 用户实现 |

### 1.3.5 爬虫后端 (8 种)

| 爬虫 | 说明 | 适用场景 |
|------|------|---------|
| **BeautifulSoup** (默认) | HTML 解析 | 静态网页 |
| **Browser (Selenium)** | 浏览器渲染 | JS 渲染页面 |
| **NoDriver** | 无驱动浏览器 | 反爬绕过 |
| **PyMuPDF** | PDF 解析 | PDF 文档 |
| **FireCrawl** | API 爬虫 | 结构化提取 |
| **Tavily Extract** | API 提取 | Tavily 生态 |
| **WebBaseLoader** | LangChain 加载 | 通用加载 |
| **ArXiv** | 学术论文 | 论文全文 |

### 1.3.6 嵌入提供商 (17 家)

| 提供商 | 默认模型 |
|--------|---------|
| OpenAI | text-embedding-3-small |
| Azure OpenAI | text-embedding-3-small |
| Cohere | embed-v4 |
| Google VertexAI | text-embedding-004 |
| Google GenAI | text-embedding-004 |
| Ollama | mxbai-embed-large |
| Together | togethercomputer/m2-bert |
| MistralAI | mistral-embed |
| HuggingFace | sentence-transformers/all-MiniLM-L6-v2 |
| Nomic | nomic-embed-text |
| VoyageAI | voyage-3 |
| DashScope | text-embedding-v3 |
| Bedrock | amazon.titan-embed |
| AIMLAPI | text-embedding-3-small |
| GigaChat | Embeddings |
| Custom | 自定义 |
| OpenRouter | 路由 |

### 1.3.7 前端技术栈

| 技术 | 用途 |
|------|------|
| NextJS 14 | 生产级前端框架 |
| Tailwind CSS | 样式系统 |
| TypeScript | 类型安全 |
| WebSocket | 实时通信 |
| Service Worker | PWA 支持 |
| Rollup | 组件库构建 |
| Google Analytics | 分析追踪 |

### 1.3.8 基础设施与工具链

| 类别 | 技术 |
|------|------|
| 容器化 | Docker, Docker Compose |
| 编排 | ECS Fargate (AWS) |
| IaC | Terraform |
| CI/CD | GitHub Actions |
| 镜像仓库 | AWS ECR |
| 监控 | LangSmith Tracing |
| 日志 | Loguru, 标准 logging |
| 包管理 | Poetry, pip, npm |
| 测试 | pytest, pytest-asyncio |
| 代码质量 | ESLint, TypeScript |
| 代理 | Nginx |

---

## 1.4 项目架构风格与理由

### 1.4.1 架构风格总览

GPT Researcher 采用**混合架构风格**，融合了多种模式：

```
┌─────────────────────────────────────────────────────┐
│                  混合架构风格                         │
├─────────────────────────────────────────────────────┤
│  分层架构 (Layered)                                  │
│    └─ Frontend → Backend API → Agent Core → External │
├─────────────────────────────────────────────────────┤
│  插件化/微内核 (Plugin/Microkernel)                   │
│    └─ Retriever/Scraper/LLM Provider 可插拔          │
├─────────────────────────────────────────────────────┤
│  面向智能体 (Agent-Oriented)                         │
│    └─ Planner + Executor + Publisher 三角色          │
├─────────────────────────────────────────────────────┤
│  事件驱动 (Event-Driven)                             │
│    └─ WebSocket 流式传输                             │
├─────────────────────────────────────────────────────┤
│  状态机 (State Machine)                              │
│    └─ LangGraph StateGraph 多 Agent 协作             │
├─────────────────────────────────────────────────────┤
│  管道-过滤器 (Pipes & Filters)                       │
│    └─ 上下文压缩管道                                 │
└─────────────────────────────────────────────────────┘
```

### 1.4.2 分层架构 (Layered Architecture)

**设计决策**: 系统明确划分为四层，每层仅依赖下层：

```
Layer 4: 前端展示层 (NextJS / Static HTML)
    ↕ WebSocket / HTTP
Layer 3: API 网关层 (FastAPI Routes + WebSocket Manager)
    ↕ 函数调用
Layer 2: Agent 编排层 (GPTResearcher + Skills)
    ↕ 接口调用
Layer 1: 基础设施层 (Retriever / Scraper / LLM / VectorStore)
```

**理由**:
- **关注点分离**: 每层职责清晰，前端变化不影响 Agent 逻辑
- **可测试性**: 每层可独立单元测试
- **可替换性**: 前端可从静态 HTML 升级到 NextJS 而不影响后端

### 1.4.3 插件化架构 (Plugin Architecture)

**设计决策**: 检索器、爬虫、LLM 提供商均采用插件化注册：

```python
# 检索器注册表 (retrievers/__init__.py)
RETRIEVER_REGISTRY = {
    "tavily": TavilySearch,
    "duckduckgo": Duckduckgo,
    "google": GoogleSearch,
    # ... 20+ 实现
}

# LLM 提供商注册表 (llm_provider/generic/base.py)
PROVIDER_REGISTRY = {
    "openai": ChatOpenAI,
    "anthropic": ChatAnthropic,
    "ollama": ChatOllama,
    # ... 27 实现
}
```

**理由**:
- **开闭原则**: 新增提供商无需修改现有代码
- **社区贡献**: 第三方可以轻松贡献新的检索器/LLM 集成
- **运行时选择**: 用户通过配置切换实现，无需代码变更

### 1.4.4 面向智能体架构 (Agent-Oriented)

**设计决策**: 采用 Planner-Executor-Publisher 三角色模型：

| 角色 | 职责 | 实现 |
|------|------|------|
| **Planner** | 生成研究问题/子话题 | `plan_research_outline()` |
| **Executor** | 并行收集信息 | `ResearchConductor.conduct_research()` |
| **Publisher** | 聚合发现为报告 | `ReportGenerator.write_report()` |

**理由**:
- **并行化**: Executor 可并行处理多个子问题，显著提速
- **关注点分离**: 规划与执行解耦，各自可独立优化
- **可观测性**: 每个阶段可独立监控和日志记录

### 1.4.5 事件驱动架构 (Event-Driven)

**设计决策**: 通过 WebSocket 实现实时流式传输：

```python
# 事件类型
EVENT_TYPES = [
    "logs",           # 研究日志
    "images",         # 图像数据
    "cost",           # 成本更新
    "research_report" # 最终报告
]
```

**理由**:
- **实时反馈**: 用户可实时观察研究进度
- **长任务友好**: 深度研究可能耗时数分钟，流式传输避免超时
- **用户体验**: 渐进式展示结果，减少等待焦虑

### 1.4.6 状态机架构 (State Machine)

**设计决策**: 多 Agent 协作使用 LangGraph StateGraph：

```python
# multi_agents/agents/orchestrator.py
workflow = StateGraph(ResearchState)
workflow.add_node("browser", research.run_initial_research)
workflow.add_node("planner", editor.plan_research)
workflow.add_node("researcher", editor.run_parallel_research)
workflow.add_node("writer", writer.run)
workflow.add_node("fact_checker", fact_checker.run)
workflow.add_node("visualizer", visualizer.run)
workflow.add_node("publisher", publisher.run)
workflow.add_node("human", human.review_plan)
```

**理由**:
- **复杂编排**: 多 Agent 协作需要精确的状态管理
- **人机协同**: Human-in-the-loop 需要条件分支
- **可恢复性**: 状态机支持检查点和恢复

---

## 1.5 关键功能特性

### 1.5.1 报告类型 (7 种)

| 报告类型 | 枚举值 | 说明 | 典型耗时 |
|---------|--------|------|---------|
| **研究报告** | `research_report` | 标准综合报告 | ~2 分钟 |
| **详细报告** | `detailed_report` | 深度分析，子话题分解 | ~5 分钟 |
| **资源报告** | `resource_report` | 资源列表与描述 | ~2 分钟 |
| **大纲报告** | `outline_report` | 结构化大纲 | ~1 分钟 |
| **自定义报告** | `custom_report` | 用户自定义格式 | 可变 |
| **子话题报告** | `subtopic_report` | 特定子话题深度报告 | ~2 分钟 |
| **深度研究** | `deep` | 递归探索，广度×深度 | ~5–15 分钟 |

### 1.5.2 数据来源 (6 种)

| 来源类型 | 说明 |
|---------|------|
| **Web** | 联网搜索 + 网页抓取 |
| **Local** | 本地文档目录 |
| **Azure** | Azure Blob Storage |
| **LangChain Documents** | LangChain 文档对象 |
| **Hybrid** | Web + 本地混合 |
| **LangChain VectorStore** | 外部向量存储 |

### 1.5.3 文章语气 (19 种)

| 语气 | 说明 |
|------|------|
| Objective | 公正无偏见 |
| Formal | 学术规范 |
| Analytical | 批判性分析 |
| Persuasive | 说服性 |
| Informative | 信息丰富 |
| Explanatory | 解释性 |
| Descriptive | 描述性 |
| Critical | 批判性 |
| Comparative | 比较性 |
| Speculative | 推测性 |
| Reflective | 反思性 |
| Narrative | 叙事性 |
| Humorous | 幽默 |
| Optimistic | 乐观 |
| Pessimistic | 悲观 |
| Simple | 简单 |
| Casual | 随意 |
| ... | 更多 |

### 1.5.4 输出格式

| 格式 | 说明 |
|------|------|
| Markdown | 默认格式 |
| PDF | 通过 md2pdf/WeasyPrint |
| Word (DOCX) | 通过 python-docx/htmldocx |
| JSON | API 响应 |

### 1.5.5 高级功能

1. **智能图像抓取与过滤**: 自动从网页提取相关图像，嵌入报告
2. **AI 图像生成**: 集成 Google Gemini (Nano Banana) 生成插图
3. **JavaScript 渲染**: 通过 Selenium 抓取 JS 渲染页面
4. **记忆与上下文**: 全程保持研究记忆和上下文连贯
5. **来源审查 (Source Curation)**: LLM 评估来源可信度
6. **MCP 集成**: 通过 Model Context Protocol 接入外部工具
7. **多 Agent 协作**: LangGraph 驱动的多角色协作
8. **聊天问答**: 基于报告的交互式问答

---

## 1.6 非功能性需求

### 1.6.1 性能 (Performance)

| 指标 | 目标 | 实现方式 |
|------|------|---------|
| 标准报告生成 | < 3 分钟 | 并行检索 + 异步 I/O |
| 深度报告生成 | < 15 分钟 | 并发控制 + 缓存 |
| 并发爬虫 | 15 Worker (可配) | WorkerPool + Semaphore |
| 单请求响应 | < 100ms (API) | FastAPI 异步 |
| 流式延迟 | < 500ms | WebSocket 实时推送 |

**关键实现**:
- `asyncio.gather()` 并行执行多个检索/抓取任务
- `WorkerPool` 控制并发度，避免资源耗尽
- `GlobalRateLimiter` 全局限速，防止 API 超限
- `asyncio.to_thread()` 将阻塞 HTTP 调用卸载到线程池

### 1.6.2 扩展性 (Scalability)

| 维度 | 策略 |
|------|------|
| **水平扩展** | 无状态 API 服务，可多副本部署 |
| **垂直扩展** | Worker 数量可配置 |
| **插件扩展** | 新 Retriever/Scraper/LLM 即插即用 |
| **配置扩展** | 环境变量 + JSON 配置文件 |
| **功能扩展** | Skill 类继承体系 |

### 1.6.3 安全性 (Security)

| 层面 | 措施 |
|------|------|
| **API Key** | 环境变量注入，不硬编码 |
| **输入验证** | Pydantic 模型验证 |
| **CORS** | 可配置允许来源 |
| **Docker** | 非 root 用户运行 |
| **依赖安全** | Brotli 版本约束 (CVE-2025-6176) |
| **MCP 路径** | 允许根路径白名单 |
| **Prompt 注入** | JSON 解析容错 + 正则回退 |

### 1.6.4 可用性 (Availability)

| 策略 | 实现 |
|------|------|
| **重试机制** | LLM 调用最多 10 次重试 |
| **降级策略** | Strategic → Smart → Fast LLM 降级 |
| **错误隔离** | 单 URL 抓取失败不影响整体 |
| **超时控制** | Tavily 请求 100s 超时 |
| **健康检查** | Docker restart: always |

### 1.6.5 可维护性 (Maintainability)

| 实践 | 说明 |
|------|------|
| **模块化** | 每个功能独立模块/包 |
| **类型注解** | 全代码类型提示 |
| **文档字符串** | 所有公共函数含 docstring |
| **日志** | 结构化日志 + JSON 研究日志 |
| **测试** | 73 个测试文件，覆盖核心路径 |
| **配置** | 集中配置管理 (Config 类) |
| **枚举** | 类型安全的枚举定义 |

### 1.6.6 可观测性 (Observability)

| 维度 | 工具/方式 |
|------|---------|
| **日志** | Loguru + 标准 logging |
| **追踪** | LangSmith Tracing (可选) |
| **成本** | 实时 token 用量与费用追踪 |
| **进度** | WebSocket 实时推送研究步骤 |
| **JSON 日志** | 结构化研究事件日志 |

---

## 1.7 项目结构概览

```
gpt-researcher/
├── gpt_researcher/          # 核心 Agent 包
│   ├── agent.py             # 主 Agent 类 (794 行)
│   ├── prompts.py           # 提示词管理 (903 行)
│   ├── actions/             # 原子操作
│   ├── skills/              # 技能模块
│   ├── retrievers/          # 检索后端 (20+)
│   ├── scraper/             # 爬虫后端 (8)
│   ├── llm_provider/        # LLM 适配器 (27)
│   ├── mcp/                 # MCP 集成
│   ├── context/             # 上下文压缩
│   ├── memory/              # 嵌入管理
│   ├── vector_store/        # 向量存储
│   ├── config/              # 配置管理
│   └── utils/               # 工具函数
├── backend/                 # 后端服务
│   ├── server/              # FastAPI 服务
│   ├── chat/                # 聊天 Agent
│   ├── report_type/         # 报告类型实现
│   └── memory/              # 后端记忆
├── multi_agents/            # 多 Agent 协作
├── frontend/                # 前端
│   ├── nextjs/              # NextJS 前端
│   └── static/              # 静态 HTML 前端
├── deep_agents/             # 深度 Agent 基准
├── docs/                    # 文档站
├── terraform/               # IaC
├── evals/                   # 评估框架
└── tests/                   # 测试 (73 文件)
```

---

## 1.8 版本演进关键节点

| 版本/时间 | 里程碑 |
|----------|-------|
| 2023-09 | 项目启动，基础研究 Agent |
| 2023-11 | OpenAI Assistant 集成 |
| 2024-05 | 迁移到 LangGraph |
| 2024-09 | 混合研究 (Hybrid Research) |
| 2025-02 | 深度研究 (Deep Research) |
| 2025-03 | 故事化报告 |
| 2026-03 | AG2 (AutoGen) 集成 |
| v0.14.7 | 当前版本，MCP 全面集成 |

---

## 1.9 总结

GPT Researcher 是一个**高度模块化、可扩展的深度研究智能体框架**。其核心价值在于：

1. **多源聚合**: 20+ 检索后端确保信息全面性
2. **插件架构**: 所有组件可插拔，易于扩展
3. **多 LLM 支持**: 27 家提供商，无供应商锁定
4. **实时流式**: WebSocket 提供卓越用户体验
5. **深度研究**: 递归探索实现超越浅层搜索
6. **多 Agent 协作**: LangGraph 驱动复杂研究任务
7. **MCP 集成**: 通过标准协议接入外部工具

项目架构清晰，代码质量良好，测试覆盖充分，是一个生产级 AI Agent 系统的优秀范例。

---

> **下一章**: → `02-c4-architecture.md` — C4 架构模型

---

# ❤️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../res/promotion.jpg)