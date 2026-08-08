# Scira 架构深度分析

> **项目**: [zaidmukaddam/scira](https://github.com/zaidmukaddam/scira)
> **定位**: 生产级 AI 搜索引擎（Perplexity 开源替代）
> **规模**: 350 个 TS/TSX 文件，~123,000 行代码
> **技术栈**: Next.js 16 · React 19 · AI SDK 5.0 · Drizzle ORM · PostgreSQL · Redis · Better Auth · Cloudflare R2 · Upstash

---

## 目录

1. [项目概览](#1-项目概览)
2. [技术栈全景](#2-技术栈全景)
3. [系统架构总览](#3-系统架构总览)
4. [核心 API 路由层](#4-核心-api-路由层)
5. [AI Provider 与模型注册系统](#5-ai-provider-与模型注册系统)
6. [搜索模式与工具系统](#6-搜索模式与工具系统)
7. [Extreme Search 深度研究引擎](#7-extreme-search-深度研究引擎)
8. [Canvas 生成式 UI 系统](#8-canvas-生成式-ui-系统)
9. [Build Mode 云端沙箱](#9-build-mode-云端沙箱)
10. [MCP 协议集成](#10-mcp-协议集成)
11. [Lookout 定时搜索](#11-lookout-定时搜索)
12. [数据库设计](#12-数据库设计)
13. [认证与支付系统](#13-认证与支付系统)
14. [架构哲学与设计亮点](#14-架构哲学与设计亮点)

---

## 1. 项目概览

Scira（原名 MiniPerplx）是一个功能极其丰富的开源 AI 搜索引擎，由 Zaid Mukaddam 开发和维护。它不仅仅是一个简单的"Perplexity 克隆"，而是在搜索体验、模型覆盖度、研究深度和商业化方面都做到了生产级水准。

**核心能力一览：**

- **100+ 个 LLM 模型**：通过 AI SDK Gateway 路由到 20+ 个 Provider（OpenAI、Anthropic、Google、xAI、DeepSeek、ZhipuAI、Moonshot、ByteDance 等）
- **13 种搜索模式**：Chat / Web / Academic / YouTube / Reddit / GitHub / X / Stocks / Crypto / Code / Extreme / MCP / Build
- **Extreme Search**：多步深度研究引擎，集成 Daytona 云沙箱执行 Python 代码、生成图表、RAG 文件检索
- **Canvas 生成式 UI**：LLM 输出 JSON Spec，前端渲染为交互式数据仪表盘（图表、表格、指标卡、时间线）
- **Build Mode**：基于 Upstash Box 的云端编码沙箱，Claude Code Agent 自主完成编码任务
- **MCP 支持**：用户可连接外部 MCP Server（GitHub、Notion、Slack 等），支持 OAuth 认证
- **Lookout**：定时搜索任务（QStash Cron 驱动），自动执行并邮件通知
- **多层级订阅**：Free / Pro / Max，通过 Polar + Dodo Payments 双支付系统

---

## 2. 技术栈全景

```
┌──────────────────────────────────────────────────────────────┐
│                        前端 (React 19)                        │
│  Next.js 16 App Router · Tailwind CSS · Shadcn UI · Recharts │
│  @json-render/react (Canvas) · Cambio (图片缩放) · streamdown │
├──────────────────────────────────────────────────────────────┤
│                     API 路由层 (Next.js)                      │
│  /api/search · /api/lookout · /api/mcp/* · /api/files /*     │
├──────────────────────────────────────────────────────────────┤
│                      AI 编排层 (AI SDK 5.0)                   │
│  streamText · tool() · UIMessageStream · Gateway 路由         │
│  ai-resumable-stream (断线重连) · better-all (并行)          │
├──────────────────────────────────────────────────────────────┤
│                    工具与 Agent 系统                          │
│  26+ 搜索工具 · Extreme Search Agent · Build Agent (Box)     │
│  MCP Client · Connectors · Auto Router                       │
├──────────────────────────────────────────────────────────────┤
│                      数据与基础设施                           │
│  PostgreSQL (Drizzle ORM) · Redis (Upstash) · R2 (Cloudflare)│
│  QStash (Upstash) · Box (Upstash) · Daytona · Polar · Dodo   │
├──────────────────────────────────────────────────────────────┤
│                      认证与安全                               │
│  Better Auth · Polar Plugin · Dodo Plugin · Session Cache    │
│  OAuth (GitHub/Google/Twitter/Microsoft) · MCP OAuth         │
└──────────────────────────────────────────────────────────────┘
```

**关键依赖：**

| 领域 | 包名 | 用途 |
|------|------|------|
| AI 编排 | `ai` (AI SDK 5.0) | streamText、tool 定义、消息转换 |
| AI 网关 | `@ai-sdk/gateway` | 多 Provider 统一路由 |
| 流式传输 | `ai-resumable-stream` | 断线重连的 UI 消息流 |
| 并行执行 | `better-all` | Promise.all 的增强版（容错 + 并发控制） |
| 搜索 API | `exa-js` / `parallel-web` / `@mendable/firecrawl-js` | 三选一的 Web 搜索后端 |
| 云沙箱 | `@upstash/box` | Build Mode 的云端编码环境 |
| 云沙箱 | `@daytonaio/sdk` | Extreme Search 的代码执行环境 |
| 向量检索 | `@vectorstores/core` | Extreme Search 内的文件 RAG |
| 对象存储 | `@aws-sdk/client-s3` | Cloudflare R2（图表、构建产物） |
| 数据库 | `drizzle-orm` | PostgreSQL ORM |
| 认证 | `better-auth` | 用户认证 + 社交登录 |
| 支付 | `@polar-sh/better-auth` / `@dodopayments/better-auth` | 双支付系统集成 |
| 定时任务 | Upstash QStash | Lookout 定时搜索调度 |
| 嵌入 | `@ai-sdk/cohere` | embed-v4.0 向量嵌入 |

---

## 3. 系统架构总览

```
用户请求 (Next.js Chat UI)
    │
    ├── /api/search ──────────────────────────────────────────────┐
    │    │                                                         │
    │    ├── 认证检查 (Better Auth Session)                        │
    │    ├── 权限检查 (Pro/Max 订阅)                               │
    │    ├── Group Config 加载 (搜索模式 → 工具集 + Prompt)       │
    │    ├── Auto Router (可选: LLM 选择最佳模型)                 │
    │    ├── MCP Tools 加载 (Pro 用户)                             │
    │    ├── Tool Loader (动态 import 工具)                        │
    │    │                                                         │
    │    └── streamText() ────────────────────────────┐           │
    │         │  model: gateway.route(model_id)        │           │
    │         │  tools: { web_search, extreme_search   │           │
    │         │           x_search, github_search ... }│           │
    │         │  stopWhen: stepCountIs(5)              │           │
    │         │                                        │           │
    │         ├── 工具调用: web_search                 │           │
    │         │    └── Exa/Parallel/Firecrawl API      │           │
    │         ├── 工具调用: extreme_search             │           │
    │         │    └── Daytona 沙箱 + Python + RAG     │           │
    │         ├── 工具调用: build (Box Agent)          │           │
    │         │    └── Upstash Box + Claude Code       │           │
    │         └── 文本/推理流式输出 → UI               │           │
    │                                                  │           │
    ├── /api/lookout ───────────────────────────────────┤  其他    │
    │    └── QStash Cron → 极端搜索 → 邮件通知          │  API     │
    │                                                    │  路由    │
    ├── /api/mcp/callback ───────────────────────────────┤         │
    │    └── MCP OAuth 回调                              │         │
    └── /api/files/* ────────────────────────────────────┘         │
         └── 文件上传到 R2                                           │
                                                                   │
    数据层 ◄────────────────────────────────────────────────────────┘
    ├── PostgreSQL: user, chat, message, subscription, lookout,
    │                user_mcp_server, build_session, usage_tracking
    ├── Redis: 断线重连流缓冲
    └── R2: 图表 PNG、构建产物下载
```

---

## 4. 核心 API 路由层

### 4.1 `/api/search/route.ts` — 主搜索引擎（1,759 行）

这是整个系统最核心的文件，处理所有用户的搜索和对话请求。流程极度优化，使用了大量并行化技术：

```
请求进入
  │
  ├── 1. 身份验证 (lightweightUser: 仅 userId/email/isProUser/isMaxUser)
  │      └── 并行启动: fullUserPromise (完整用户查询，后续 await)
  │
  ├── 2. 模型权限检查
  │      ├── requiresMaxSubscription → Max 用户专属模型
  │      ├── requiresProSubscription → Pro 用户专属模型
  │      └── MCP/Extreme 模式 → 需要 Pro
  │
  ├── 3. 并行初始化 (better-all)
  │      ├── getGroupConfig(group) → 工具列表 + System Prompt
  │      ├── getCustomInstructions → 用户自定义指令
  │      ├── getUserPreferences → 搜索 Provider、模型偏好等
  │      ├── initializeChatAndChecks → 聊天初始化 + 用量检查
  │      └── persistedMessages → 从 DB 加载历史消息
  │
  ├── 4. 消息处理
  │      ├── 消息水合 (DB 历史 + 新消息)
  │      ├── 用户消息持久化 (同步 await)
  │      ├── 消息剪裁 (>10 条用户消息时 prune)
  │      └── 文件附件提取 (PDF/CSV/DOCX/XLSX)
  │
  ├── 5. Auto Router (可选)
  │      └── LLM 根据用户查询选择最佳模型
  │
  ├── 6. MCP 工具加载 (Pro 用户, MCP/Extreme 模式)
  │      └── resolveUserMcpTools → 动态工具集
  │
  ├── 7. streamText()
  │      ├── tools: loadConfiguredTools() 动态加载
  │      ├── stopWhen: stepCountIs(5) [MCP 模式: 50]
  │      ├── maxRetries: 10
  │      ├── experimental_transform: markdownJoinerTransform
  │      └── providerOptions: gateway 路由 + 模型特定参数
  │
  └── 8. 后台异步 (after())
         ├── AI 生成标题
         ├── 保存 assistant 消息
         ├── 用量计数
         └── 关闭 MCP 连接
```

**关键设计模式：**

1. **lightweightUser 快速认证** — 不等待完整用户查询，仅从 Session 缓存获取最小信息集，节省 ~50ms
2. **better-all 并行** — 所有独立的初始化操作（config、instructions、preferences、chat init、messages）并行执行
3. **持久化消息水合** — 新消息和数据库历史消息合并，避免传输冗余数据
4. **消息剪裁策略** — 超过 10 条用户消息时，保留最近 5 条的完整工具调用，旧消息仅保留推理文本
5. **gateway 多 Provider 路由** — 每个模型可以指定 `order`（优先 Provider）和 `only`（允许的 Provider 列表）

### 4.2 `/api/lookout/route.ts` — 定时搜索（1,022 行）

Lookout 是 Scira 的"订阅式搜索"功能，用户设定一个搜索查询和频率（每日/每周/每月），系统会自动在指定时间执行搜索并发送邮件通知。

```
QStash Cron 触发
  │
  ├── 从 DB 获取 Lookout 配置 (prompt, frequency, searchMode)
  ├── 创建新 Chat 记录
  ├── 加载对应模式的全部工具
  ├── streamText() 执行搜索
  │    └── truncateMarkdown() 安全截断（避免断链接/代码块）
  ├── 计算下次运行时间 (CronExpressionParser)
  ├── 更新 QStash 调度
  └── sendLookoutCompletionEmail()
```

---

## 5. AI Provider 与模型注册系统

### 5.1 Provider 注册（`ai/providers.ts`，20,832 字符）

Scira 使用 `@ai-sdk/gateway` 作为统一网关，注册了 20+ 个 Provider：

```typescript
export const scira = createGateway({
  providers: {
    openai: openai,          // GPT-5/4.1/o3/o4 系列
    anthropic: anthropic,    // Claude Sonnet/Opus 4.6
    google: google,          // Gemini 2.5/3 Pro
    xai: xai,                // Grok 4/4.1
    deepseek: deepseek,      // DeepSeek V3
    zai: zhipu,              // 智谱 GLM-4.6/4.7
    moonshotai: moonshot,    // Kimi K2/K2.5
    bytedance: bytedance,    // 豆包
    alibaba: alibaba,        // 通义千问
    minimax: minimax,
    fireworks: fireworks,    // 推理加速
    baseten: baseten,
    bedrock: bedrock,        // AWS Nova
    streamlake: streamlake,
    novita: novita,
    togetherai: togetherai,
    // ... 更多
  },
});
```

`shouldBypassRateLimits()` 函数根据用户订阅级别返回是否跳过速率限制。

### 5.2 模型注册（`ai/models.ts`，14,671 字符）

定义了 100+ 个模型，每个模型包含：

```typescript
{
  id: 'scira-gpt5',
  name: 'GPT-5',
  description: 'OpenAI 的旗舰模型',
  group: 'chat',              // 搜索模式
  requiresProSubscription: true,
  visionEnabled: true,
  reasoningEnabled: true,
  maxOutputTokens: 16384,
}
```

模型按 `group` 分组，每个 group 对应一种搜索模式。`scira-auto` 是 Auto Router 的特殊 ID，`scira-default` 是后备模型。

### 5.3 Auto Router（`lib/search/auto-router.ts`）

用户可以选择 `scira-auto`，系统会用 LLM 根据用户查询自动选择最佳模型：

```typescript
// 用户自定义路由规则
const routes = [
  { name: 'coding',     description: '编程、代码相关问题',     model: 'scira-gpt-5.1-codex' },
  { name: 'research',   description: '学术研究、论文',          model: 'scira-anthropic-think' },
  { name: 'creative',   description: '创意写作、头脑风暴',     model: 'scira-gpt5' },
  { name: 'vision',     description: '图片分析',               model: 'scira-glm-4.6v' },
];

// LLM 分析查询 → 匹配路由 → 返回模型
```

如果路由失败或有图片但选中模型不支持视觉，回退到 `scira-default`。

---

## 6. 搜索模式与工具系统

### 6.1 Group Config（`lib/search/group-config.ts`，31,908 字符）

这是工具系统的核心调度逻辑。每种搜索模式（group）定义了：

```typescript
{
  tools: ['web_search', 'retrieve'],          // 激活的工具列表
  instructions: 'You are a helpful search assistant...', // System Prompt
}
```

共 13 个 group：`chat`、`web`、`academic`、`youtube`、`reddit`、`github`、`x`、`stocks`、`crypto`、`code`、`extreme`、`mcp`、`build`。

### 6.2 Tool Loader（`lib/search/tool-loader.ts`）

动态加载工具，使用 `await import()` 按需加载，避免初始包过大：

```typescript
switch (toolName) {
  case 'web_search':
    const { webSearchTool } = await import('@/lib/tools/web-search');
    tools.web_search = webSearchTool(dataStream, searchProvider);
    break;
  case 'extreme_search':
    const { extremeSearchTool } = await import('@/lib/tools/extreme-search');
    tools.extreme_search = extremeSearchTool(
      dataStream, contextFiles, extremeSearchModel, mcpDynamicTools
    );
    break;
  // ... 26+ 工具
}
```

### 6.3 工具清单

| 工具名 | 数据源 | 说明 |
|--------|--------|------|
| `web_search` | Exa / Parallel / Firecrawl | 三选一的 Web 搜索，支持去重 |
| `academic_search` | Exa (academic mode) | 学术论文搜索 |
| `youtube_search` | YouTube Data API | 视频搜索 |
| `reddit_search` | Reddit API | Reddit 帖子搜索 |
| `github_search` | GitHub API | 仓库/代码搜索 |
| `x_search` | X (Twitter) API | 社交媒体搜索 |
| `stock_chart` | Financial APIs | 股票图表 |
| `currency_converter` | Exchange Rate API | 货币转换 |
| `coin_data` / `coin_ohlc` | CoinGecko | 加密货币数据 |
| `get_weather_data` | OpenWeather | 天气查询 |
| `text_translate` | Google Translate | 文本翻译 |
| `track_flight` | Flight Tracker API | 航班追踪 |
| `find_place_on_map` / `nearby_places_search` | Google Places | 地图搜索 |
| `movie_or_tv_search` / `trending_movies` / `trending_tv` | TMDB | 影视搜索 |
| `spotify_search` | Spotify API | 音乐搜索 |
| `code_interpreter` | 内置 | 代码执行 |
| `code_context` | GitHub | 代码上下文 |
| `retrieve` | Firecrawl | URL 内容抓取 |
| `prediction_search` | Metaculus | 预测市场搜索 |
| `file_query_search` | VectorStores (Cohere embed) | 文件 RAG 检索 |
| `search_memories` / `add_memory` | Supermemory | 用户记忆 |
| `connectors_search` | Connectors | 连接服务搜索 |
| `extreme_search` | Daytona + Exa + Python | 多步深度研究 |
| `greeting` / `datetime` | 内置 | 时间/问候 |

### 6.4 Web Search 策略模式（`lib/tools/web-search.ts`）

```typescript
// 策略接口
interface SearchStrategy {
  search(queries, options): Promise<{ searches, results }>;
}

// 三个策略实现
class ParallelSearchStrategy implements SearchStrategy { ... }  // Parallel AI
class ExaSearchStrategy implements SearchStrategy { ... }       // Exa
class FirecrawlSearchStrategy implements SearchStrategy { ... } // Firecrawl

// 用户偏好选择 provider
const strategy = searchProvider === 'exa'
  ? new ExaSearchStrategy(exa)
  : searchProvider === 'firecrawl'
    ? new FirecrawlSearchStrategy(firecrawl)
    : new ParallelSearchStrategy(parallel, firecrawl);
```

搜索结果经过**域名去重**（`deduplicateByDomainAndDomain`）和**标题清理**（移除方括号/圆括号内容）。

---

## 7. Extreme Search 深度研究引擎

`lib/tools/extreme-search.ts`（1,963 行）是 Scira 最复杂的工具——一个完整的多步研究 Agent。

### 7.1 工作流程

```
extremeSearch(researchPrompt)
  │
  ├── 1. LLM 规划研究
  │      └── generateText() → 分解为多个子查询
  │
  ├── 2. 并行搜索每个子查询
  │      ├── Exa neural search (语义搜索)
  │      ├── Firecrawl 抓取完整页面内容
  │      └── X (Twitter) 实时数据
  │
  ├── 3. 文件 RAG (如果用户上传了文件)
  │      ├── PDFReader / CSVReader / DocxReader / ExcelReader
  │      ├── VectorStoreIndex.fromDocuments()
  │      └── cohere embed-v4.0 → retriever.search()
  │
  ├── 4. Daytona 云沙箱执行 Python
  │      ├── 数据分析 (numpy/pandas/scipy)
  │      ├── 图表生成 (matplotlib/seaborn/plotly)
  │      ├── 金融数据 (yfinance)
  │      └── PDF 处理 (pdfplumber/pymupdf)
  │      → 图表 PNG 上传到 R2
  │
  ├── 5. Token 计数 (tokenc)
  │      └── 精确控制上下文窗口
  │
  └── 6. rerank 重排序 (Cohere)
         └── 从所有收集的来源中选出最相关的
```

### 7.2 可用的 Python 库

```python
numpy, pandas, matplotlib, scipy, scikit-learn, yfinance,
requests, uv, seaborn, plotly, sympy, pydantic, regex,
PyPDF2, pdfplumber, pymupdf, tabula-py, httpx, aiohttp,
urllib3, beautifulsoup4, lxml, scrapy, selenium
```

### 7.3 极端搜索模型选择

用户可以选择 7 个极端搜索专用模型（`scira-ext-1` 到 `scira-ext-8`），通过 `ExtremeSearchModelId` 类型约束。

---

## 8. Canvas 生成式 UI 系统

Scira 的 Canvas 是一个**LLM 驱动的生成式 UI 仪表盘系统**，允许 LLM 输出结构化 JSON Spec，前端渲染为交互式数据可视化组件。

### 8.1 三层架构

```
lib/canvas/
├── catalog.ts   — 组件目录定义 (Zod schema)
├── registry.tsx — React 组件实现 (Shadcn UI + Recharts)
└── renderer.tsx — Spec → React 树渲染器
```

### 8.2 Catalog — 组件目录（347 行）

使用 `@json-render/core` 定义组件 Schema，LLM 可以输出以下组件：

**布局组件：**
- `Stack` — Flex 布局容器（水平/垂直，间距控制）
- `Card` — 卡片容器（标题 + 描述 + 内容槽）
- `Grid` — 响应式网格（1-3 列）

**排版组件：**
- `Heading` — 标题（h1-h4）
- `Text` — 文本（支持 muted 样式）
- `Separator` — 视觉分隔线

**数据展示：**
- `Metric` — 指标卡（标签 + 值 + 趋势箭头 + 详情）
- `KPIRow` — 2-4 个指标的横向横幅
- `Table` — 数据表格（支持 `$state` 绑定）
- `Badge` — 状态徽章（4 种 variant）
- `Alert` — 警告/信息提示
- `StatComparison` — A/B 对比（带 delta）
- `LayerCard` — 分层卡片（标签 + 标题 + 链接）
- `SourceCard` — 来源引用卡（favicon + 域名 + 摘要）
- `Link` — 胶囊形链接（favicon + 域名）

**图表：**
- `BarChart` — 柱状图（支持多系列 yKeys）
- `LineChart` — 折线图
- `PieChart` — 饼图/环形图

**内容组件：**
- `Callout` — 高亮提示框（info/tip/warning/important）
- `Accordion` — 可折叠手风琴
- `Timeline` — 垂直时间线
- `Quote` — 引用块（带作者和来源）
- `Image` — 图片（仅限 R2 生成的图表）

### 8.3 Registry — React 实现（1,022 行）

每个 Catalog 组件都有对应的 React 实现，使用 Tailwind CSS + Shadcn UI + Recharts：

```tsx
// 示例: Metric 组件
Metric: ({ props }) => {
  const TrendIcon = props.trend === 'up' ? TrendingUp 
                  : props.trend === 'down' ? TrendingDown : Activity;
  // 数字动画: 解析 value 中的数字部分，用 CSS count-up 动画
  return (
    <div className="rounded-lg border ...">
      <p className="text-[10px] uppercase">{props.label}</p>
      <p className="text-sm font-semibold tabular-nums">
        <span className="canvas-count-up" style={{ '--target': numValue }} />
      </p>
    </div>
  );
}
```

图表组件支持：多系列柱状图、聚合函数（sum/count/avg）、自动标签角度调整、颜色自动分配、状态绑定（`$state` 路径引用）。

### 8.4 Renderer — 渲染器

```tsx
export function CanvasRenderer({ spec, loading }) {
  return (
    <StateProvider initialState={spec.state}>
      <VisibilityProvider>
        <ActionProvider>
          <div className={loading ? "" : "canvas-stagger"}>
            <Renderer spec={spec} registry={registry} fallback={fallback} />
          </div>
        </ActionProvider>
      </VisibilityProvider>
    </StateProvider>
  );
}
```

- `StateProvider` — 提供 `$state` 路径绑定（如 `{ "$state": "/results" }`）
- `VisibilityProvider` — 组件可见性控制
- `ActionProvider` — 交互事件处理（Canvas 模式下为空，纯展示）
- `canvas-stagger` — CSS 入场动画（交错出现效果）
- `sanitizeSpec` — 确保 children 字段是数组

---

## 9. Build Mode 云端沙箱

`lib/tools/build-tools.ts`（1,003 行）实现了基于 Upstash Box 的云端编码 Agent。

### 9.1 BoxManager — 沙箱生命周期管理

```typescript
class BoxManager {
  // 懒创建 + 重连机制
  async getBox(): Promise<Box> {
    if (this.box) return this.box;
    if (this.existingBoxId) {
      // 重连已有 Box (resume)
      this.box = await Box.get(this.existingBoxId);
      await box.resume();
    } else {
      // 创建新 Box
      this.box = await this.initBox();
    }
  }

  // 初始化: 加载用户 MCP servers + 预装 skills
  async initBox() {
    return Box.create({
      runtime: this.runtime,  // node/python/golang/ruby/rust
      agent: {
        model: ClaudeCode.Sonnet_4_6,
        runner: Agent.ClaudeCode,
      },
      skills: [
        'vercel-labs/skills/find-skills',
        'anthropics/skills/frontend-design',
        'vercel-labs/agent-skills/vercel-react-best-practices',
        'vercel-labs/agent-skills/web-design-guidelines',
        'shubhamsaboo/awesome-llm-apps/python-expert',
        'fastapi/fastapi/fastapi',
      ],
      mcpServers: [
        { name: 'web-search', package: '@anthropic/mcp-web-search' },
        ...userMcpServers,  // 用户的 MCP servers
      ],
    });
  }
}
```

### 9.2 四个核心工具

1. **`box_exec`** — 在沙箱内执行 shell 命令
   ```typescript
   const run = await box.exec.command(command);
   // 流式输出 execId, command, status, stdout 到前端
   ```

2. **`box_write_file`** — 写文件到沙箱文件系统
   ```typescript
   await box.files.write({ path, content });
   ```

3. **`box_download`** — 从沙箱下载文件到 R2
   ```typescript
   // 目录 → 自动 zip
   // 文件 → base64 → R2 上传 → 返回公开 URL
   const url = `${R2_PUBLIC_URL}/scira/builds/${nanoid()}/${filename}`;
   ```

4. **`box_agent`** — 委托任务给 Claude Code Agent
   ```typescript
   const run = await box.agent.stream({
     prompt: skillPreamble + prompt,
     onToolUse: (tool) => emitToolCall(tool.name, tool.input),
   });
   // 流式: text-delta / reasoning / tool-call / finish
   // 返回: result, cost (inputTokens, outputTokens, totalUsd, computeMs)
   ```

### 9.3 Vercel 集成

如果用户连接了 Vercel MCP，Agent 会自动读取 OAuth token 并支持部署：

```
/workspace/home/.box-internal/mcp-config.json
→ 提取 Bearer token
→ export VERCEL_TOKEN=<token>
→ vercel --token $VERCEL_TOKEN --prod
```

---

## 10. MCP 协议集成

### 10.1 数据模型（`userMcpServer` 表）

每个用户可以注册多个 MCP Server，支持四种认证方式：

```typescript
authType: 'none' | 'bearer' | 'header' | 'oauth'
```

OAuth 类型的 Server 存储完整的 OAuth 流程信息：
- `oauthIssuerUrl` / `oauthAuthorizationUrl` / `oauthTokenUrl`
- `oauthClientId` / `oauthClientSecretEncrypted`
- `oauthAccessTokenEncrypted` / `oauthRefreshTokenEncrypted`
- `oauthAccessTokenExpiresAt` / `oauthConnectedAt`

敏感字段全部加密存储（`encryptedCredentials`），支持工具级别的禁用（`disabledTools`）。

### 10.2 MCP Client（`lib/tools/mcp-client.ts`）

在 `/api/search` 中，Pro 用户在 MCP 或 Extreme 模式下会动态加载 MCP 工具：

```typescript
if (shouldLoadMcpTools && lightweightUser) {
  const { resolveUserMcpTools } = await import('@/lib/tools/mcp-client');
  const resolvedMcp = await resolveUserMcpTools({
    userId: lightweightUser.userId,
    dataStream,
  });
  mcpDynamicTools = resolvedMcp.tools;
  closeMcpTools = resolvedMcp.closeAll;
}
```

MCP 工具与内置工具合并后传给 `streamText()`，LLM 可以自由调用。

### 10.3 认证头解析（`lib/mcp/auth-headers.ts`）

```typescript
// 根据认证类型返回请求头
async function resolveMcpAuthHeaders({ server, userId }) {
  switch (server.authType) {
    case 'none': return {};
    case 'bearer': return { Authorization: `Bearer ${decrypt(token)}` };
    case 'header': return JSON.parse(decrypt(server.encryptedCredentials));
    case 'oauth':
      // 检查 token 过期 → 自动刷新
      if (isExpired(server.oauthAccessTokenExpiresAt)) {
        const newToken = await refreshOAuthToken(server);
        await updateUserMcpServer(...);
        return { Authorization: `Bearer ${newToken.accessToken}` };
      }
      return { Authorization: `Bearer ${decrypt(server.oauthAccessTokenEncrypted)}` };
  }
}
```

---

## 11. Lookout 定时搜索

Lookout 是 Scira 的"AI 订阅搜索"功能——用户设定一个查询和频率，系统定期执行搜索并发送邮件。

### 11.1 数据模型（`lookout` 表）

```typescript
{
  title: string,
  prompt: string,                    // 搜索查询
  frequency: 'once' | 'daily' | 'weekly' | 'monthly' | 'yearly',
  cronSchedule: string,              // cron 表达式
  timezone: string,                  // 用户时区
  nextRunAt: timestamp,
  qstashScheduleId: string,          // QStash 调度 ID
  status: 'active' | 'paused' | 'archived' | 'running',
  searchMode: 'extreme' | 'web' | 'academic' | ...,
  runHistory: Array<{                // JSON 运行历史
    runAt: string,
    chatId: string,
    status: 'success' | 'error' | 'timeout',
    duration?: number,
    tokensUsed?: number,
    searchesPerformed?: number,
  }>,
}
```

### 11.2 执行流程

```
QStash Cron 触发 /api/lookout
  │
  ├── 获取 Lookout 配置
  ├── 验证用户订阅状态
  ├── 创建新 Chat
  ├── 加载全部搜索工具（非流式）
  ├── streamText() 执行搜索
  │    └── stopWhen: stepCountIs(30) — 允许更多步骤
  ├── truncateMarkdown() — 安全截断结果
  ├── 更新 nextRunAt + QStash 调度
  ├── 更新 runHistory
  └── sendLookoutCompletionEmail()
```

`truncateMarkdown` 函数是一个精心设计的 Markdown 安全截断器，它会：
- 跟踪代码围栏状态（`` ``` ``）
- 跟踪内联代码状态（`` ` ``）
- 跟踪链接文本/URL 嵌套深度
- 在段落边界、句子结尾、换行符处优先截断
- 避免断开链接、列表项或代码块

---

## 12. 数据库设计

Scira 使用 PostgreSQL + Drizzle ORM，共 16 张表：

### 12.1 核心表结构

```
user ──┬── session (会话)
       ├── account (OAuth 账号)
       ├── verification (验证码)
       ├── chat ──┬── message (消息，含 parts JSON)
       │          └── stream (断线重连流)
       ├── extremeSearchUsage (极端搜索用量，月度)
       ├── messageUsage (消息用量，日度)
       ├── anthropicUsage (Anthropic 用量，日度)
       ├── googleUsage (Google 用量，日度)
       ├── agentModeUsageEvents (Agent 用量，append-only)
       ├── customInstructions (自定义指令)
       ├── userPreferences (偏好设置 JSON)
       ├── subscription (Polar 订阅)
       ├── payment (Dodo 支付记录)
       ├── dodosubscription (Dodo 订阅)
       ├── lookout (定时搜索)
       ├── userMcpServer (MCP 服务器)
       └── buildSession (构建会话)
```

### 12.2 设计亮点

1. **Chat 使用 UUIDv7** — 时间有序的主键，利于索引和分页
2. **Message.parts 为 JSON** — 存储多模态消息部件（text/reasoning/tool_call/sources/ui_spec）
3. **用量追踪多表分离** — 不同 Provider 有不同的重置周期（Anthropic 日度、Extreme 月度、Message 日度）
4. **agentModeUsageEvents 是 append-only** — 以 messageId 为幂等键，防止删除会话绕过用量限制
5. **userPreferences 是 JSON** — 灵活的偏好存储，支持搜索 Provider、模型排序、Auto Router 配置等
6. **双订阅系统** — `subscription`（Polar）和 `dodosubscription`（Dodo Payments）并存

### 12.3 索引策略

每张表都有精心设计的索引：
- 复合索引：`(userId, createdAt)`、`(userId, status)`、`(userId, isPinned, updatedAt)`
- 唯一索引：`(userId, date)` 用于用量的原子 upsert
- 查询模式索引：`chatId`、`userId` 等外键字段

---

## 13. 认证与支付系统

### 13.1 Better Auth 配置（`lib/auth.ts`，556 行）

```typescript
export const auth = betterAuth({
  appName: 'scira',
  database: drizzleAdapter(maindb, { provider: 'pg', schema: {...} }),
  socialProviders: {
    github: {...},
    google: {...},
    twitter: {...},
    microsoft: {...},
  },
  plugins: [
    dash(),              // Better Auth 仪表盘
    lastLoginMethod(),   // 记住上次登录方式
    polar({...}),        // Polar 支付集成
    dodopayments({...}), // Dodo Payments 集成
  ],
  databaseHooks: {
    session: {
      delete: {
        before: async (session) => {
          // 登出时立即从缓存驱逐 token（不等 TTL 过期）
          invalidateSessionCacheForToken(session.token);
        },
      },
    },
  },
});
```

### 13.2 双支付系统

Scira 同时集成了两个支付 Provider：

**Polar（Starter 层）：**
- Webhook 处理：`subscription.created/active/canceled/revoked/uncanceled/updated`
- 产品 ID 验证：防止不匹配的订阅
- 用户关联：通过 `customer.externalId` 关联

**Dodo Payments（Pro + Max 层）：**
- Webhook 处理：`subscription.active/cancelled/expired/failed`
- 自动互斥：如果用户升级到 Max，自动撤销 Polar 订阅
- 完整的订阅生命周期管理

### 13.3 订阅层级

| 层级 | 价格 | 功能 |
|------|------|------|
| Free | $0 | 基础搜索、有限模型 |
| Starter (Polar) | 低价 | 更多模型、扩展搜索 |
| Pro (Dodo) | 中价 | Extreme Search、MCP、文件 RAG |
| Max (Dodo) | 高价 | 全部模型（含 Claude Opus、GPT-5.2）、最高用量 |

---

## 14. 架构哲学与设计亮点

### 14.1 "最大化利用 AI SDK 生态"

与 Vane 的"一切自研"不同，Scira 深度依赖 AI SDK 5.0 生态：

- `streamText` + `tool()` → Agent 编排
- `@ai-sdk/gateway` → 多 Provider 路由
- `createUIMessageStream` → 流式传输
- `ai-resumable-stream` → 断线重连
- `pruneMessages` → 上下文窗口管理
- `convertToModelMessages` → 消息格式转换

Scira 的自研重心在**工具实现**和**用户体验**上，而非基础设施。

### 14.2 "搜索策略" 模式

Web Search 使用经典的策略模式，三个搜索 Provider 可互换：

```typescript
interface SearchStrategy {
  search(queries, options): Promise<SearchResult>;
}
```

用户可以在偏好设置中选择 `exa`、`parallel` 或 `firecrawl`，无需改动其他代码。

### 14.3 极致的并行化

`/api/search` 中几乎所有初始化操作都是并行的：

```typescript
const { criticalResult, config, customInstructionsResult,
        chatInitResult, userPreferencesResult, persistedMessages } =
  await all({
    async criticalResult() { return criticalChecksPromise; },
    async config() { return configPromise; },
    async customInstructionsResult() { return customInstructionsPromise; },
    async chatInitResult() { return chatInitializationPromise; },
    async userPreferencesResult() { return userPreferencesPromise; },
    async persistedMessages() { return persistedMessagesPromise; },
  }, getBetterAllOptions());
```

使用 `better-all` 而非原生 `Promise.all`，提供了更好的错误处理和并发控制。

### 14.4 "轻量认证" 策略

```typescript
// 快速路径: 仅从 Session 缓存获取最小信息
const lightweightUser = await getLightweightUser(session);
// { userId, email, isProUser, isMaxUser } — 无额外 DB 查询

// 慢速路径: 并行启动完整用户查询（后续按需 await）
const fullUserPromise = getFullUser(session.userId);
```

这减少了首字节延迟（TTFB），尤其对于重度用户。

### 14.5 Canvas vs Morphic Generative UI

两个项目都使用 `@json-render/react`，但 Scira 的 Canvas 更专注于**研究仪表盘**：

| 特性 | Scira Canvas | Morphic Generative UI |
|------|-------------|----------------------|
| 组件数 | 20+ | 5 (Grid/Image/Button/Related/Sources) |
| 图表 | ✅ Bar/Line/Pie (Recharts) | ❌ |
| 数据表格 | ✅ (TanStack Table) | ❌ |
| 指标卡 | ✅ Metric/KPIRow/StatComparison | ❌ |
| 时间线 | ✅ | ❌ |
| 手风琴 | ✅ | ❌ |
| 状态绑定 | ✅ `$state` 路径 | ❌ |
| 交互动作 | ❌ (纯展示) | ✅ (按钮点击) |

Scira Canvas 是"研究报告渲染器"，Morphic 是"搜索结果增强器"。

### 14.6 用量追踪的防绕过设计

`agentModeUsageEvents` 表使用 append-only 设计，以 `messageId` 为幂等键：

```typescript
uniqueIndex('agentModeUsageEvents_messageId_unique').on(table.messageId)
```

即使用户删除会话或聊天记录，用量事件仍然存在，防止通过删除数据来重置用量限制。

### 14.7 markdownJoinerTransform

`experimental_transform: markdownJoinerTransform()` 是一个流式 Markdown 合并器，将多个工具调用的输出片段合并成连贯的 Markdown 文本，处理跨片段的格式问题。

---

## 总结

Scira 是一个工程复杂度极高的 AI 搜索引擎，其核心特色在于：

1. **模型覆盖最广** — 100+ 个模型通过 Gateway 路由，Auto Router 自动选择
2. **研究深度最强** — Extreme Search 集成云沙箱 Python 执行 + RAG + 图表生成
3. **生成式 UI 最丰富** — Canvas 系统支持 20+ 种数据可视化组件
4. **商业化最完整** — 双支付系统、多层级订阅、精细化用量追踪
5. **扩展性最强** — MCP 协议支持、Connectors 系统、26+ 内置工具

它站在 AI SDK 生态的肩膀上，将自研精力集中在**工具实现**、**用户体验**和**商业化**三个维度，是一个优秀的"产品级 AI 搜索引擎"参考实现。

---

*文档生成时间: 2026-07-31*
*分析基于: scira 主分支 (commit at clone time)*
*代码规模: 350 TS/TSX 文件, ~123,000 行*

---

☕️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../res/promotion.jpg)