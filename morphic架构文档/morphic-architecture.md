# Morphic 架构文档

> **项目**: [miurla/morphic](https://github.com/miurla/morphic)
> **版本**: v1.5.0 | **License**: Apache-2.0
> **定位**: AI 驱动的搜索引擎，带生成式 UI（Generative UI）

---

## 目录

1. [项目概述](#1-项目概述)
2. [架构总览](#2-架构总览)
3. [核心数据模型](#3-核心数据模型)
4. [Agent 与工具系统](#4-agent-与工具系统)
5. [Generative UI 系统](#5-generative-ui-系统)
6. [搜索提供商架构](#6-搜索提供商架构)
7. [流式响应管线](#7-流式响应管线)
8. [认证与数据库设计](#8-认证与数据库设计)
9. [前端组件架构](#9-前端组件架构)
10. [部署与配置](#10-部署与配置)
11. [架构亮点与改进方向](#11-架构亮点与改进方向)
12. [关键文件索引](#12-关键文件索引)

---

## 1. 项目概述

### 1.1 一句话定义

Morphic 是一个基于 **Vercel AI SDK** 构建的 AI 搜索引擎，核心创新是 **Generative UI**——LLM 在回答中直接输出 JSON Spec 渲染富 UI 组件（图片网格、交互按钮），超越纯 Markdown。

### 1.2 核心特性

| 特性 | 说明 |
|------|------|
| 🎨 **Generative UI** | LLM 输出 JSON Spec（RFC 6902 JSON Patch），实时渲染 Grid/Image/Button/Heading 组件 |
| 🔍 **两种搜索模式** | quick（快速，~5 步）/ adaptive（深度，~20 步） |
| 🤖 **Vercel AI SDK 5.0** | 使用 ToolLoopAgent 原生 agent 循环，流式工具调用 |
| 🔎 **5 个搜索提供商** | Tavily（默认）/ SearXNG / Brave / Exa / Firecrawl |
| 🧠 **6 个 AI 提供商** | OpenAI / Anthropic / Google / Ollama / Vercel Gateway / OpenAI-Compatible |
| 📝 **Todo 工具** | adaptive 模式下 LLM 可创建任务列表跟踪复杂研究 |
| 🔐 **Supabase Auth** | 完整认证系统，支持访客模式 |
| 🗄️ **PostgreSQL + RLS** | 行级安全策略，数据库层隔离用户数据 |
| 📊 **可观测性** | Langfuse + OpenTelemetry 全链路追踪 |
| 📎 **文件上传** | S3 兼容存储 |

### 1.3 技术栈

```
前端:    Next.js 16 + React 19 + Tailwind CSS + shadcn/ui (Radix)
AI:     Vercel AI SDK 5.0-alpha (ToolLoopAgent + streamText + smoothStream)
搜索:    Tavily / SearXNG / Brave / Exa / Firecrawl
数据库:  PostgreSQL 17 + Drizzle ORM
认证:    Supabase Auth
缓存:    Redis (Upstash / 本地)
追踪:    Langfuse + OpenTelemetry
渲染:    @json-render/react + streamdown
运行时:  Bun (Node.js 22.x)
```

### 1.4 代码规模

- **313 个 TypeScript/TSX 文件**，约 **37,300 行代码**
- 后端核心（`lib/`）：~120 个文件
- API 路由（`app/api/`）：8 个端点
- React 组件（`components/`）：~60 个组件

---

## 2. 架构总览

### 2.1 系统架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                      用户浏览器                                   │
│              React 19 SPA (Next.js App Router)                   │
│   ┌────────────┐  ┌───────────┐  ┌──────────────────────────┐  │
│   │ Chat Panel  │  │ Artifact  │  │ Generative UI (json-     │  │
│   │ (消息输入)   │  │ (结果展示) │  │ render: Grid/Image/Btn) │  │
│   └─────┬──────┘  └─────┬─────┘  └──────────────────────────┘  │
└─────────┼───────────────┼──────────────────────────────────────┘
          │               │ SSE Stream (AI SDK UIMessageStream)
          ▼               ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Next.js API Routes                             │
│  POST /api/chat ───────────────────────────────────────────┐    │
│    • Auth (Supabase) → Guest? → Rate Limit → Mode Select   │    │
│    • createChatStreamResponse() / createEphemeral...()      │    │
└────┼───────────────────────────────────────────────────────┼────┘
     │                                                        │
     ▼                                                        │
┌─────────────────────────────────────────────────────────┐    │
│              Vercel AI SDK: ToolLoopAgent                │    │
│                                                          │    │
│  Researcher Agent (maxSteps: quick=20 / adaptive=50)     │    │
│    ├── search tool ──→ SearchProvider ─────────────┐     │    │
│    ├── fetch tool ──→ Jina/Tavily/Regular HTML     │     │    │
│    ├── askQuestion (前端确认工具)                   │     │    │
│    └── todoWrite (仅 adaptive 模式)                 │     │    │
│                                                     │     │    │
│  LLM 流式生成 Markdown + ```spec JSON Patch ───────┼─────┘    │
└─────────────────────────────────────────────────────┼──────────┘
                                                      │
                          ┌───────────────────────────┘
                          ▼
              ┌───────────────────────┐
              │  Search Provider      │
              │  Factory              │
              │  ┌────────────────┐   │
              │  │ Tavily (API)   │   │
              │  │ SearXNG (self) │   │
              │  │ Brave (API)    │   │
              │  │ Exa (neural)   │   │
              │  │ Firecrawl      │   │
              │  └────────────────┘   │
              └───────────────────────┘
```

### 2.2 请求处理流程

```
用户提问
    │
    ├── ① 认证 & 限流
    │     getCurrentUserId() → guest? → rate limit check
    │
    ├── ② 模式选择 (Cookie)
    │     searchMode = 'quick' | 'adaptive'
    │
    ├── ③ 消息准备
    │     prepareMessages() → stripSpec → stripReasoning (OpenAI)
    │     → convertToModelMessages → pruneMessages → truncate
    │
    ├── ④ Researcher Agent
    │     ToolLoopAgent.stream({ messages, tools, stopWhen })
    │     ├── LLM 决策调用 search/fetch/todoWrite
    │     ├── 工具流式 yield 状态（searching → complete）
    │     └── smoothStream({ chunking: 'word' }) 平滑输出
    │
    ├── ⑤ 流式响应
    │     toUIMessageStreamResponse() → SSE
    │     前端逐步渲染：工具调用 → 研究过程 → Markdown 回答
    │     → ```spec 块 → Generative UI 组件
    │
    └── ⑥ 持久化 (onFinish)
          persistStreamResults() → PostgreSQL
          (消息、Parts、标题、traceId)
```

---

## 3. 核心数据模型

### 3.1 数据库 Schema（PostgreSQL + Drizzle）

Morphic 使用 5 张表，全部启用 **RLS（行级安全）**：

```
chats ──1:N── messages ──1:N── parts
  │
  ├──1:N── notes
  └──1:N── libraryFiles (上传文件)

feedback (独立表)
```

**chats 表**：

```sql
id          VARCHAR(191) PRIMARY KEY   -- cuid2
title       TEXT NOT NULL
userId      VARCHAR(255) NOT NULL
visibility  VARCHAR(256) DEFAULT 'private'  -- public | private
createdAt   TIMESTAMP DEFAULT NOW()
-- RLS: 用户只能操作 user_id = current_user_id 的行
```

**messages 表**：

```sql
id          VARCHAR(191) PRIMARY KEY
chatId      VARCHAR(191) → chats.id (CASCADE)
role        VARCHAR(256)        -- user | assistant | ...
metadata    JSONB               -- { traceId, searchMode, modelId }
createdAt   TIMESTAMP
updatedAt   TIMESTAMP
```

**parts 表（核心设计——单表多态）**：

这是 Morphic 最精妙的数据库设计。一条消息的完整内容（文本、推理、工具调用、文件、来源）全部拆成有序的 Parts：

```sql
id             VARCHAR(191) PRIMARY KEY
messageId      → messages.id (CASCADE)
order          INTEGER NOT NULL        -- 排序
type           VARCHAR(256) NOT NULL   -- 见下方类型枚举

-- Text parts
text_text      TEXT

-- Reasoning parts（思考链）
reasoning_text TEXT

-- File parts
file_mediaType  VARCHAR(256)
file_filename   VARCHAR(1024)
file_url        TEXT
file_key        TEXT

-- Source URL parts（引用来源）
source_url_sourceId  VARCHAR(256)
source_url_url       TEXT
source_url_title     TEXT

-- Source document parts
source_document_title    TEXT
source_document_filename VARCHAR(1024)
source_document_url      TEXT
source_document_snippet  TEXT

-- Tool parts（工具调用）
tool_toolCallId    VARCHAR(256)
tool_state         VARCHAR(256)   -- input-streaming | input-available | output-available | output-error
tool_search_input  JSON
tool_search_output JSON
tool_fetch_input   JSON
tool_fetch_output  JSON
tool_question_input/output JSON
tool_todoWrite_input/output JSON
tool_dynamic_input/output/name/type JSON  -- MCP 和运行时工具

-- Data parts（Generative UI spec）
data_prefix   VARCHAR(256)
data_content  JSON          -- spec JSON Patch 数据
data_id       VARCHAR(256)
```

**CHECK 约束** 保证数据完整性：

```sql
CHECK (type != 'text' OR text_text IS NOT NULL)
CHECK (type != 'reasoning' OR reasoning_text IS NOT NULL)
CHECK (tool_state IN ('input-streaming', 'input-available', 'output-available', 'output-error'))
```

### 3.2 Part 类型枚举

```
text           → 普通文本
reasoning      → LLM 推理过程（思考链）
file           → 上传文件
source-url     → URL 引用来源
source-document → 文档引用来源
tool-search    → 搜索工具调用
tool-fetch     → 网页抓取工具调用
tool-question  → 澄清问题工具
tool-todoWrite → 任务管理工具
tool-dynamic   → MCP / 运行时动态工具
data           → Generative UI spec 数据
```

### 3.3 SearchResults 类型

```typescript
type SearchResults = {
  results: { title: string; content: string; url: string; score?: number }[];
  images: { url: string; title?: string; description?: string; sourceUrl?: string }[];
  query: string;
  number_of_results: number;
  toolCallId?: string;
}
```

---

## 4. Agent 与工具系统

### 4.1 Researcher Agent

`lib/agents/researcher.ts`

Morphic **不自己实现 Agent 循环**，而是使用 Vercel AI SDK 5.0 的 `ToolLoopAgent`：

```typescript
const agent = new ToolLoopAgent({
  model: getModel(model),              // 从 Provider Registry 获取
  instructions: systemPrompt,          // 模式相关 Prompt
  tools,                                // 搜索/抓取/提问/任务工具
  activeTools: activeToolsList,         // 当前模式启用的工具子集
  stopWhen: stepCountIs(maxSteps),     // 步数限制
  experimental_telemetry: { ... }      // Langfuse 追踪
})
```

**两种模式配置**：

| | Quick 模式 | Adaptive 模式 |
|---|---|---|
| maxSteps | 20 | 50 |
| 启用工具 | search, fetch | search, fetch, todoWrite |
| 搜索类型 | 强制 `type="optimized"` | 保留原样 |
| Prompt 目标 | ~5 步内完成 | ~20 步深度研究 |

**Quick 模式的特殊优化**：`wrapSearchToolForQuickMode` 包装搜索工具，强制使用 `type: 'optimized'`（直接返回内容片段，不需额外 fetch）。

### 4.2 工具系统

#### 4.2.1 Search Tool

`lib/tools/search.ts`

```typescript
createSearchTool(model) → tool({
  description: getSearchToolDescription(),
  inputSchema: getSearchSchemaForModel(model),
  async *execute({ query, type, max_results, search_depth, ... }) {
    yield { state: 'searching', query }   // 流式状态
    const provider = createSearchProvider(searchAPI)
    const result = await provider.search(query, ...)
    yield { state: 'complete', ...result }  // 最终结果
  },
  toModelOutput: ({ output }) => {
    // 从工具输出中删除 citationMap（与 results 重复）和 state（流式标记）
    // 保留 images（prompt 需要嵌入 URL）和 toolCallId（引用需要）
  }
})
```

#### 4.2.2 Fetch Tool

`lib/tools/fetch.ts`

支持三种抓取模式：

```
regular → 直接 HTTP fetch + HTML 解析（快，10s 超时）
api     → Jina Reader API（Markdown 格式，支持 PDF）
       → 或 Tavily Extract API（fallback）
```

#### 4.2.3 Question Tool

`lib/tools/question.ts`

特殊设计——**没有 execute 函数**，让前端处理（AI SDK 的前端确认机制）：

```typescript
tool({
  description: 'Ask a clarifying question...',
  inputSchema: getQuestionSchemaForModel(model)
  // execute: undefined → 前端渲染交互式 UI
})
```

#### 4.2.4 Todo Tool

`lib/tools/todo.ts`

会话级任务管理，仅在 adaptive 模式启用：

```typescript
// 闭包内维护 sessionTodos（每个 agent 实例独立）
let sessionTodos: TodoItem[] = []

todoWrite execute:
  - 更新 sessionTodos
  - 返回 { completedCount, totalCount, todos }
```

### 4.3 Search Mode Prompts

`lib/agents/prompts/search-mode-prompts.ts`

两种模式的 Prompt 差异显著，共同特点：

**引用格式强制**：`[number](#toolCallId)` 格式，严格的放置规则（句号后）

**Source Direction**：支持 `include_domains` / `exclude_domains` 引导搜索来源

**Early Stop Criteria**：4 个停止条件（可回答、~70% 重叠、收益递减、足够覆盖）

**Generative UI 指令**：嵌入 `getImageSpecPrompt()` 和 `getRelatedQuestionsSpecPrompt()`

---

## 5. Generative UI 系统

这是 Morphic **最核心的创新**。

### 5.1 设计理念

传统 AI 搜索引擎只输出 Markdown 文本。Morphic 让 LLM 在回答中嵌入 ` ```spec ` 代码块，包含 **RFC 6902 JSON Patch** 操作，前端实时渲染为交互式组件。

### 5.2 Spec 架构

`lib/render/schema.ts`

```typescript
spec: {
  root: string,                    // 根元素 ID
  elements: Record<string, {
    type: 'Heading' | 'Stack' | 'Button' | 'Grid' | 'Image',
    props: { ... },                // Zod 校验的属性
    children: string[],            // 子元素 ID 列表
    on?: { press: { action, params } }  // 事件处理
  }>
}
```

### 5.3 组件目录

`lib/render/catalog.ts`

| 组件 | 属性 | 用途 |
|------|------|------|
| `Heading` | title, icon? | 标题标签 |
| `Stack` | direction?, gap? | 布局容器（垂直/水平） |
| `Button` | text, variant?, icon? | 可点击按钮 |
| `Grid` | columns(1-4), gap? | 固定列网格 |
| `Image` | src, sourceUrl?, title?, aspectRatio? | 内联图片 |

**Action**：`submitQuery`（点击后提交新查询）

### 5.4 Spec 在 LLM Prompt 中的注入

`lib/render/prompt.ts`

两个 Spec Prompt 函数：

**`getImageSpecPrompt()`**：指示 LLM 在回答中嵌入图片组

```
规则：
- 只使用搜索结果中的真实 URL（不编造）
- 2-4 张图片/组，放在上下文相关位置
- ```spec 块包含 JSONL（每行一个 JSON Patch 操作）
- 用 Grid 包裹 Image 组件
```

**`getRelatedQuestionsSpecPrompt()`**：在回答末尾输出相关问题

```
规则：
- 只在有后续价值时生成（跳过问候/简单问题）
- 恰好 3 个问题：深化 / 行动 / 拓展
- 用 Button variant="link" 渲染
- 点击触发 submitQuery action
```

### 5.5 渲染管线

```
LLM 输出 Markdown
    │
    ├── 普通文本 → streamdown 渲染（Markdown → HTML）
    │
    └── ```spec 块 → streamdown custom renderer
         │
         ├── parse-spec-block.ts: 解析 JSONL
         ├── @json-render/react: 执行 JSON Patch，构建组件树
         ├── registry.tsx: 映射 type → React 组件
         └── 渲染 Grid/Image/Button/Heading
```

**关键文件**：

- `lib/render/parse-spec-block.ts`：解析 spec JSONL
- `lib/render/strip-spec-blocks.ts`：从消息历史中剥离 spec（避免影响 LLM 上下文）
- `lib/render/streamdown-spec.tsx`：streamdown 自定义渲染器
- `lib/render/migrations.ts`：spec 版本迁移

---

## 6. 搜索提供商架构

### 6.1 Provider 抽象

`lib/tools/search/providers/base.ts`

```typescript
interface SearchProvider {
  search(
    query: string,
    maxResults: number,
    searchDepth: 'basic' | 'advanced',
    includeDomains: string[],
    excludeDomains: string[],
    options?: {
      type?: 'general' | 'optimized'
      content_types?: Array<'web' | 'video' | 'image' | 'news'>
    }
  ): Promise<SearchResults>
}
```

### 6.2 工厂模式

`lib/tools/search/providers/index.ts`

```typescript
function createSearchProvider(type?: SearchProviderType): SearchProvider {
  switch (type) {
    case 'tavily':   return new TavilySearchProvider()
    case 'searxng':  return new SearXNGSearchProvider()
    case 'brave':    return new BraveSearchProvider()
    case 'exa':      return new ExaSearchProvider()
    case 'firecrawl': return new FirecrawlSearchProvider()
    default:         return new TavilySearchProvider()
  }
}
```

### 6.3 五个提供商对比

| Provider | 类型 | 特点 | 图片支持 |
|----------|------|------|----------|
| **Tavily** | 商业 API | 默认提供商，返回内容片段 + 图片描述 | ✅ 含 sourceUrl 映射 |
| **SearXNG** | 自托管 | 元搜索聚合，Docker 内置 | ❌ |
| **Brave** | 商业 API | 支持 content_types（web/video/image/news） | ✅ 多媒体最佳 |
| **Exa** | 商业 API | 神经搜索（语义匹配） | ❌ |
| **Firecrawl** | 商业 API | 网页抓取 + 内容提取 | ❌ |

### 6.4 Advanced Search（SearXNG）

SearXNG 的 `advanced` 深度搜索走独立的 API 端点 `/api/advanced-search`，使用 Redis 缓存：

```
searchTool execute
  └── if searxng + advanced:
        fetch('/api/advanced-search', { query, maxResults, ... })
        → 后端路由用 Redis 缓存 SearXNG 结果
```

---

## 7. 流式响应管线

### 7.1 AI SDK UIMessageStream

Morphic 使用 Vercel AI SDK 的原生流式协议：

```typescript
researchAgent.stream({
  messages: modelMessages,
  abortSignal,
  experimental_transform: smoothStream({ chunking: 'word' })
})

// 转换为 UI 消息流
result.toUIMessageStreamResponse({
  messageMetadata: ({ part }) => {
    if (part.type === 'start') return { traceId, searchMode, modelId }
  },
  onFinish: async ({ responseMessage, isAborted }) => {
    await persistStreamResults(...)
  },
  onError: (error) => serializePublicError(error)
})
```

### 7.2 smoothStream

`smoothStream({ chunking: 'word' })` 是 AI SDK 的流式平滑转换器，按单词分块输出，避免逐字符渲染的闪烁。

### 7.3 上下文窗口管理

Morphic 有完善的上下文管理链：

```
prepareMessages() → stripSpecFromMessages() → stripReasoningParts() (OpenAI)
  → convertToModelMessages() → pruneMessages() → shouldTruncate? → truncateMessages()
```

**pruneMessages**：AI SDK 内置，保留最近消息的推理和工具调用，删除历史中的冗余
**truncateMessages**：基于模型 context window 限制截断旧消息

### 7.4 持久化

`lib/streaming/helpers/persist-stream-results.ts`

流完成后异步持久化到 PostgreSQL：

- 将 UIMessage 拆解为 messages + parts 记录
- 自动生成聊天标题（新会话时并行启动）
- 存储 traceId 用于关联 Langfuse 追踪

---

## 8. 认证与数据库设计

### 8.1 Supabase Auth

- 完整的认证流程：登录、注册、OAuth、密码重置
- 访客模式（`ENABLE_GUEST_CHAT=true`）：匿名用户共享单一 user_id
- 云部署强制认证（adaptive 模式限流）

### 8.2 行级安全（RLS）

Morphic 在数据库层实现数据隔离：

```sql
-- chats 表 RLS
CREATE POLICY users_manage_own_chats ON chats
  FOR ALL USING (user_id = current_setting('app.current_user_id', true));

CREATE POLICY public_chats_readable ON chats
  FOR SELECT USING (visibility = 'public');

-- messages 表 RLS（JOIN 到 chats 验证所有权）
CREATE POLICY users_manage_chat_messages ON messages
  FOR ALL USING (
    EXISTS (SELECT 1 FROM chats
            WHERE chats.id = chat_id
            AND chats.user_id = current_setting('app.current_user_id', true))
  );

-- parts 表 RLS（三层 JOIN）
CREATE POLICY users_manage_message_parts ON parts
  FOR ALL USING (
    EXISTS (SELECT 1 FROM messages
            INNER JOIN chats ON chats.id = messages.chat_id
            WHERE messages.id = message_id
            AND chats.user_id = current_setting('app.current_user_id', true))
  );
```

### 8.3 Rate Limiting

三层限流策略：

```
Guest Limit    → IP 级别（访客）
Overall Limit  → 用户级别（所有模式）
Adaptive Limit → 用户级别（仅 adaptive 模式，云部署强制认证）
```

---

## 9. 前端组件架构

### 9.1 页面路由

```
app/
├── page.tsx              → 首页（搜索入口）
├── search/[id]/page.tsx  → 搜索结果页
├── share/[id]/page.tsx   → 分享页
├── auth/                 → 认证页面（login, signup, oauth...）
└── api/                  → API 路由
```

### 9.2 核心组件

```
components/
├── artifact/             → 搜索结果 + AI 回答展示
│   ├── artifact-root.tsx
│   ├── search-artifact-content.tsx
│   ├── reasoning-content.tsx
│   └── tool-invocation-content.tsx
├── sidebar/              → 聊天历史侧边栏
├── inspector/            → 调试面板
├── library/              → 文件库
├── ui/                   → shadcn/ui 基础组件
└── spec-fence-block.tsx  → Generative UI spec 渲染器
```

### 9.3 State Management

- **Server Components**：Next.js RSC 负责初始数据加载
- **Client Hooks**：`hooks/` 目录下的自定义 hooks
- **AI SDK React**：`useChat` / `useUIMessage` 等 hooks 管理流式状态

---

## 10. 部署与配置

### 10.1 Docker Compose 部署

```yaml
services:
  morphic:     # Next.js 应用
  postgres:    # PostgreSQL 17
  redis:       # Redis（SearXNG 缓存）
  searxng:     # SearXNG 元搜索
```

```bash
docker pull ghcr.io/miurla/morphic:latest
docker compose up -d
# 自动启动 PostgreSQL + Redis + SearXNG + Morphic
```

### 10.2 环境变量

**必需**：

```bash
OPENAI_API_KEY=       # 默认 AI 提供商
TAVILY_API_KEY=       # 默认搜索提供商（Docker 内置 SearXNG 可替代）
DATABASE_URL=         # PostgreSQL 连接
```

**可选**：

```bash
# AI 提供商
ANTHROPIC_API_KEY=
GOOGLE_GENERATIVE_AI_API_KEY=
OLLAMA_BASE_URL=
AI_GATEWAY_API_KEY=
OPENAI_COMPATIBLE_API_KEY= + OPENAI_COMPATIBLE_API_BASE_URL=

# 搜索提供商
SEARCH_API=tavily|searxng|brave|exa|firecrawl
BRAVE_API_KEY= / EXA_API_KEY= / FIRECRAWL_API_KEY=

# 功能开关
ENABLE_AUTH=true|false
ENABLE_GUEST_CHAT=true|false
ENABLE_SAVE_CHAT_HISTORY=true|false
NEXT_PUBLIC_ENABLE_SHARE=true|false

# 可观测性
LANGFUSE_API_KEY= + LANGFUSE_PUBLIC_KEY=
```

### 10.3 模型配置

`public/config/models.json`：

```json
[{
  "id": "gpt-4o",
  "provider": "OpenAI",
  "providerId": "openai",
  "enabled": true,
  "toolCallType": "native",      // native | manual
  "toolCallModel": "gpt-4o-mini" // 可选覆盖
}]
```

---

## 11. 架构亮点与改进方向

### 11.1 架构亮点

**① Generative UI——Morphic 的灵魂**

LLM 输出 JSON Spec（RFC 6902 JSON Patch），前端实时渲染富 UI。这不是简单的 Markdown 中的图片，而是一个完整的组件渲染系统。图片网格、交互按钮、相关问题——全部由 LLM 在回答中动态生成。**这是 Morphic 区别于所有其他 AI 搜索引擎的核心创新**。

**② Vercel AI SDK 原生集成**

不像 Vane 自研一切，Morphic 深度使用 AI SDK 5.0 的 ToolLoopAgent、smoothStream、pruneMessages、convertToModelMessages 等。减少自研代码，获得 SDK 升级红利。

**③ Parts 单表多态设计**

一条 AI 回答（包含文本段落、推理过程、工具调用、引用来源、Generative UI spec）被拆解为有序的 parts 记录，存储在一张表中。通过 `type` 字段和 CHECK 约束保证完整性。**查询简单，Schema 演进灵活**。

**④ 数据库层 RLS 安全**

所有 5 张表都启用了 PostgreSQL 行级安全策略。即使用户绕过应用层，数据库仍然拒绝未授权访问。三层 JOIN 策略（parts → messages → chats）确保深度嵌套数据的隔离。

**⑤ 搜索提供商策略模式**

5 个搜索提供商通过统一接口 + 工厂模式管理。新增提供商只需继承 BaseSearchProvider 并在工厂注册。支持运行时通过环境变量切换。

**⑥ 完善的上下文窗口管理**

prepareMessages → stripSpec → stripReasoning → convert → prune → truncate 的完整链条，适配不同模型的上下文限制。特别是 `stripSpecFromMessages` 在发给 LLM 之前剥离 Generative UI spec，避免 token 浪费。

### 11.2 与 Vane 的对比

| 维度 | Morphic | Vane |
|------|---------|------|
| AI 框架 | Vercel AI SDK（原生） | 自研抽象层 |
| 搜索模式 | quick / adaptive（2种） | speed / balanced / quality（3种） |
| 分类器 | 无（直接进入研究） | LLM Classifier（7个布尔决策） |
| Widget 系统 | 无（Generative UI 替代） | 天气/股票/计算器 |
| 流式协议 | AI SDK UIMessageStream | 自研 NDJSON + JSON Patch |
| 数据库 | PostgreSQL + RLS | SQLite |
| 认证 | Supabase Auth（完整） | 无 |
| Generative UI | ✅ JSON Spec 渲染 | ❌ |
| 可观测性 | Langfuse + OTel | 无 |
| 文件上传 | S3 兼容 | 本地文件系统 |
| 搜索提供商 | 5 个（策略模式） | SearxNG 单一 |
| 代码规模 | ~37K 行 | ~55K 行 |

### 11.3 潜在改进方向

**① AI SDK Alpha 版本风险**

依赖 `ai: ^6.0.180`（5.0-alpha.2）和 `@ai-sdk/react: ^3.0.182`，是 alpha 版本。API 可能变化，生产稳定性有风险。

**② Generative UI 的安全边界**

LLM 输出的 JSON Spec 直接渲染为 React 组件。虽然 `@json-render/react` 有 Schema 校验，但如果 LLM 输出恶意 URL（如 XSS payload），仍需额外的内容安全策略。

**③ Spec Prompt Token 消耗**

`getImageSpecPrompt()` 和 `getRelatedQuestionsSpecPrompt()` 合计约 3,000+ token，每次请求都注入。可考虑缓存或条件注入。

**④ SearXNG Advanced Search 的单点依赖**

Advanced search 走独立的 `/api/advanced-search` 端点 + Redis 缓存。如果 Redis 不可用，advanced search 直接失败，没有 fallback。

**⑤ Title 生成的串行依赖**

新会话的标题生成虽然有 `titlePromise` 并行启动，但 `generateChatTitle` 内部会调用 LLM，增加额外的 API 成本。

---

## 12. 关键文件索引

### Agent 系统

| 文件 | 行数 | 说明 |
|------|------|------|
| `lib/agents/researcher.ts` | 161 | Researcher Agent（ToolLoopAgent 封装） |
| `lib/agents/prompts/search-mode-prompts.ts` | 340 | Quick/Adaptive 模式 Prompt |
| `lib/agents/title-generator.ts` | - | 聊天标题自动生成 |

### 工具系统

| 文件 | 行数 | 说明 |
|------|------|------|
| `lib/tools/search.ts` | 238 | 搜索工具（流式 yield 状态） |
| `lib/tools/fetch.ts` | 198 | 网页抓取（regular/api 双模式） |
| `lib/tools/question.ts` | 18 | 澄清问题（前端确认工具） |
| `lib/tools/todo.ts` | 59 | 任务管理（仅 adaptive） |
| `lib/tools/dynamic.ts` | - | MCP / 运行时动态工具 |
| `lib/tools/search/providers/base.ts` | 51 | 搜索提供商抽象基类 |
| `lib/tools/search/providers/index.ts` | 44 | 工厂模式注册 |

### Generative UI

| 文件 | 行数 | 说明 |
|------|------|------|
| `lib/render/schema.ts` | 42 | Spec Schema 定义 |
| `lib/render/catalog.ts` | 70 | 组件目录（5 组件 + 1 action） |
| `lib/render/prompt.ts` | 116 | LLM Spec Prompt（图片+相关问题） |
| `lib/render/registry.tsx` | 25 | React 组件注册 |
| `lib/render/parse-spec-block.ts` | - | JSONL 解析器 |
| `lib/render/strip-spec-blocks.ts` | - | 从消息历史剥离 spec |
| `lib/render/streamdown-spec.tsx` | 31 | streamdown 自定义渲染器 |
| `lib/render/migrations.ts` | - | Spec 版本迁移 |
| `lib/render/components/image.tsx` | - | Image 组件 |
| `lib/render/components/grid.tsx` | - | Grid 组件 |
| `lib/render/components/button.tsx` | - | Button 组件 |
| `lib/render/components/heading.tsx` | - | Heading 组件 |
| `lib/render/components/stack.tsx` | - | Stack 组件 |

### 流式管线

| 文件 | 行数 | 说明 |
|------|------|------|
| `lib/streaming/create-chat-stream-response.ts` | 274 | 认证用户流式响应 |
| `lib/streaming/create-ephemeral-chat-stream-response.ts` | 168 | 访客流式响应 |
| `lib/streaming/helpers/prepare-messages.ts` | - | 消息预处理 |
| `lib/streaming/helpers/persist-stream-results.ts` | - | 结果持久化 |
| `lib/streaming/helpers/convert-data-part.ts` | - | Data Part 转换 |
| `lib/streaming/helpers/strip-reasoning-parts.ts` | - | OpenAI 推理剥离 |
| `lib/streaming/helpers/strip-spec-from-messages.ts` | - | Spec 剥离 |

### 数据库

| 文件 | 行数 | 说明 |
|------|------|------|
| `lib/db/schema.ts` | 394 | 5 表 Schema + RLS 策略 |
| `lib/actions/chat-db.ts` | - | 数据库操作 |
| `drizzle/` | - | 迁移文件 |

### API 路由

| 文件 | 说明 |
|------|------|
| `app/api/chat/route.ts` | 聊天 API（SSE 流式） |
| `app/api/advanced-search/route.ts` | SearXNG 高级搜索 |
| `app/api/chats/route.ts` | 聊天列表 |
| `app/api/upload/route.ts` | 文件上传 |
| `app/api/feedback/route.ts` | 用户反馈 |

### 模型配置

| 文件 | 说明 |
|------|------|
| `lib/utils/registry.ts` | Provider Registry（6 个提供商） |
| `lib/models/fetch-models.ts` | 动态获取模型列表 |
| `public/config/models.json` | 模型配置 |
| `lib/utils/model-selection.ts` | 模型选择逻辑 |

---

## 附录：Morphic vs Vane 架构哲学对比

```
Morphic 的哲学：站在 Vercel AI SDK 巨人的肩膀上
├── 信任 SDK 的 Agent 循环（ToolLoopAgent）
├── 信任 SDK 的流式协议（UIMessageStream）
├── 信任 SDK 的消息管理（pruneMessages, smoothStream）
├── 自研重心放在 Generative UI 上（核心差异化）
└── 用 PostgreSQL + Supabase 构建生产级基础设施

Vane 的哲学：一切自研，完全控制
├── 自研 LLM 抽象层（8 个 Provider）
├── 自研 Agent 编排（Classifier + Researcher + Writer）
├── 自研流式协议（NDJSON + RFC 6902 JSON Patch）
├── 自研 Widget 系统（天气/股票/计算器）
├── 三种搜索模式（speed/balanced/quality）
└── 用 SQLite 保持极简部署
```

两种路线没有对错：Morphic 更快迭代（站在 SDK 上），Vane 控制力更强。Morphic 的 Generative UI 是独特亮点，Vane 的三模式 + Widget 系统提供了不同的用户体验。

---

*文档生成时间: 2026-07-30 | 基于 Morphic v1.5.0 源码分析*

---

☕️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../res/promotion.jpg)