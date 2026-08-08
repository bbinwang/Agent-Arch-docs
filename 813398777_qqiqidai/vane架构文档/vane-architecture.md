# Vane 架构文档

> **项目**: [ItzCrazyKns/Vane](https://github.com/ItzCrazyKns/Vane)
> **版本**: v1.12.2 | **License**: MIT | **Stars**: ~35.9K
> **定位**: 隐私优先的本地 AI 搜索引擎（Perplexica 的精神继承者）

---

## 目录

1. [项目概述](#1-项目概述)
2. [架构总览](#2-架构总览)
3. [核心概念与数据模型](#3-核心概念与数据模型)
4. [核心模块详解](#4-核心模块详解)
5. [搜索流水线深度解析](#5-搜索流水线深度解析)
6. [多 LLM 提供商系统](#6-多-llm-提供商系统)
7. [流式通信协议](#7-流式通信协议)
8. [前端架构](#8-前端架构)
9. [API 接口设计](#9-api-接口设计)
10. [部署与配置](#10-部署与配置)
11. [架构亮点与改进方向](#11-架构亮点与改进方向)
12. [关键文件索引](#12-关键文件索引)

---

## 1. 项目概述

### 1.1 一句话定义

Vane 是一个 **完全本地运行的 AI 问答引擎**，结合互联网搜索与本地/云端 LLM，提供带引用来源的精准回答，同时保持搜索隐私。

### 1.2 核心特性

| 特性 | 说明 |
|------|------|
| 🔒 隐私优先 | 所有数据存储在本地，不依赖外部服务托管 |
| 🤖 多 LLM 支持 | OpenAI / Anthropic / Gemini / Groq / Ollama / LMStudio / Lemonade / Transformers（本地） |
| ⚡ 三种搜索模式 | speed（快速）/ balanced（均衡）/ quality（深度研究） |
| 🔍 多搜索源 | web（网页）/ academic（学术论文）/ discussions（社区讨论）/ personal（上传文件） |
| 🧩 Widgets 系统 | 天气、股票、计算器——与研究并行运行的即时信息卡片 |
| 📷 多媒体搜索 | 图片和视频搜索（独立端点） |
| 📄 文件上传 | PDF / Word / 纯文本，支持语义搜索 |
| 🌐 SearxNG 集成 | 元搜索引擎，聚合多个搜索引擎结果 |

### 1.3 技术栈

```
前端:  Next.js 16 + React 18 + TailwindCSS + Headless UI
后端:  Next.js API Routes (Node.js runtime)
搜索:  SearxNG (元搜索) + Playwright (网页抓取)
存储:  SQLite (Drizzle ORM) + 本地文件系统
AI:   自研 LLM 抽象层（8 个 provider）
类型:  TypeScript + Zod（schema 验证）
流式:  TransformStream + EventEmitter + RFC 6902 JSON Patch
```

### 1.4 代码规模

- **473 个 TypeScript/TSX 文件**，约 **55,000 行代码**
- 后端核心（`src/lib/`）：~95 个 .ts 文件
- API 路由（`src/app/api/`）：16 个端点
- 内嵌 `morphic/` 子模块（fork 自 miurla/morphic，提供参考实现）

---

## 2. 架构总览

### 2.1 系统架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                         用户浏览器                                │
│                    React SPA (Next.js)                           │
└──────────────────────────┬──────────────────────────────────────┘
                           │ HTTP + SSE Stream
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Next.js API Routes                            │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────┐ ┌────────┐ │
│  │/api/chat │ │/api/search│ │/api/images│ │/videos │ │/uploads│ │
│  └────┬─────┘ └────┬─────┘ └──────────┘ └────────┘ └────────┘ │
│       │            │                                           │
└───────┼────────────┼───────────────────────────────────────────┘
        │            │
        ▼            ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Agent 编排层                                   │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              SearchAgent (主编排器)                       │    │
│  │                                                          │    │
│  │  ① Classifier ──── Zod schema → 分类决策                 │    │
│  │       │                                                  │    │
│  │  ② ┌──┴──────────────────┐  (并行)                       │    │
│  │    │                     │                               │    │
│  │    ▼                     ▼                               │    │
│  │  Researcher          WidgetExecutor                      │    │
│  │  (Agentic Loop)      (Weather/Stock/Calc)                │    │
│  │    │                     │                               │    │
│  │  ③ └──┬──────────────────┘                               │    │
│  │       ▼                                                  │    │
│  │  Writer (LLM 答案生成 + 引用标注)                          │    │
│  └─────────────────────────────────────────────────────────┘    │
└──────────┬──────────────┬──────────────┬────────────────────────┘
           │              │              │
           ▼              ▼              ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│  SearxNG     │ │  Playwright  │ │  LLM Provider│
│  (元搜索)    │ │  (网页抓取)   │ │  (8 种)      │
└──────────────┘ └──────────────┘ └──────────────┘
```

### 2.2 请求处理三阶段

Vane 的核心设计是 **"分类 → 研究+Widgets 并行 → 答案生成"** 的三阶段流水线：

```
用户提问
    │
    ▼
① 分类阶段 (Classifier)
    │  • 是否需要搜索？(skipSearch)
    │  • 哪种搜索？(academic/discussion/personal)
    │  • 需要 Widgets？(weather/stock/calculation)
    │  • 生成独立追问 (standaloneFollowUp)
    │
    ├──────────────────────────────┐
    ▼                              ▼
②a 研究阶段 (Researcher)     ②b Widgets (并行)
    │  Agentic 循环：              │  • 天气 (Open-Meteo API)
    │  reasoning → search →        │  • 股票 (Yahoo Finance)
    │  reasoning → scrape →        │  • 计算器 (mathjs)
    │  ... → done                  │
    │                              │
    └──────────┬───────────────────┘
               ▼
③ 答案生成 (Writer)
    │  • 合并搜索结果 + Widget 上下文
    │  • LLM 流式生成，带 [1][2] 引用
    │  • 持久化到 SQLite
    ▼
  返回给用户
```

---

## 3. 核心概念与数据模型

### 3.1 Block 协议

Vane 的核心数据传输单元是 **Block**。每个回答由多个 Block 组成，通过流式传输增量推送：

```typescript
// src/lib/types.ts
type Block =
  | TextBlock          // { type: 'text', data: string }
  | SourceBlock        // { type: 'source', data: Chunk[] }
  | SuggestionBlock    // { type: 'suggestion', data: string[] }
  | WidgetBlock        // { type: 'widget', data: { widgetType, params } }
  | ResearchBlock;     // { type: 'research', data: { subSteps: [...] } }
```

**ResearchBlock** 是最复杂的类型，包含多种子步骤：

```
ResearchBlock
  └── subSteps[]
       ├── ReasoningResearchBlock    (推理过程)
       ├── SearchingResearchBlock    (正在搜索)
       ├── SearchResultsResearchBlock(搜索结果)
       ├── ReadingResearchBlock      (正在阅读)
       ├── UploadSearchingResearchBlock  (上传文件搜索)
       └── UploadSearchResultsResearchBlock
```

### 3.2 Chunk（信息片段）

```typescript
type Chunk = {
  content: string;           // 文本内容
  metadata: {
    title: string;
    url: string;
    similarity?: number;     // 余弦相似度
    embedding?: number[];    // 向量嵌入（去重用）
  };
};
```

### 3.3 ClassifierOutput（分类决策）

```typescript
type ClassifierOutput = {
  classification: {
    skipSearch: boolean;           // 是否跳过搜索
    personalSearch: boolean;       // 搜索上传文件
    academicSearch: boolean;       // 学术搜索
    discussionSearch: boolean;     // 社区讨论搜索
    showWeatherWidget: boolean;
    showStockWidget: boolean;
    showCalculationWidget: boolean;
  };
  standaloneFollowUp: string;      // 上下文无关的独立问题
};
```

### 3.4 数据库 Schema

Vane 使用 SQLite + Drizzle ORM，仅两张表：

```sql
-- chats 表：存储会话元数据
CREATE TABLE chats (
  id        TEXT PRIMARY KEY,
  title     TEXT NOT NULL,
  createdAt TEXT NOT NULL,
  sources   TEXT DEFAULT '[]',    -- JSON: SearchSources[]
  files     TEXT DEFAULT '[]'     -- JSON: { fileId, name }[]
);

-- messages 表：存储消息和 Block 数据
CREATE TABLE messages (
  id             INTEGER PRIMARY KEY,
  messageId      TEXT NOT NULL,
  chatId         TEXT NOT NULL,
  backendId      TEXT NOT NULL,   -- Session ID
  query          TEXT NOT NULL,
  createdAt      TEXT NOT NULL,
  responseBlocks TEXT DEFAULT '[]', -- JSON: Block[]
  status         TEXT DEFAULT 'answering' -- answering|completed|error
);
```

---

## 4. 核心模块详解

### 4.1 SearchAgent — 主编排器

`src/lib/agents/search/index.ts`

SearchAgent 是整个搜索流水线的入口，负责协调三个阶段的执行：

```
searchAsync(session, input)
  │
  ├── 1. 数据库初始化（创建/更新 message 记录）
  │
  ├── 2. 调用 Classifier 进行分类
  │
  ├── 3. 并行启动 Widget 和 Researcher
  │     ├── WidgetExecutor.executeAll() → emitBlock(widget)
  │     └── Researcher.research() (如果 !skipSearch)
  │
  ├── 4. Promise.all([widgetPromise, searchPromise])
  │
  ├── 5. 合并上下文：搜索结果 + Widget 结果
  │     构造 writerPrompt
  │
  ├── 6. LLM 流式生成答案
  │     首个 chunk → emitBlock(text)
  │     后续 chunk → updateBlock (RFC 6902 patch)
  │
  └── 7. 持久化到 DB，emit('end')
```

**关键设计**：Widget 和 Researcher 完全并行执行，通过 `Promise.all` 等待两者完成后再进入答案生成阶段。Widget 结果通过 `llmContext` 字段提供给 Writer LLM，但明确告知 LLM 不要将 Widget 结果作为引用来源。

### 4.2 Classifier — 分类器

`src/lib/agents/search/classifier.ts`

使用 `generateObject<T>` 方法让 LLM 输出结构化的分类决策：

- **输入**：聊天历史 + 用户查询 + 启用的搜索源
- **输出**：7 个布尔标志 + 1 个独立的追问字符串
- **Schema**：Zod schema 强制约束输出格式

**standaloneFollowUp 的作用**：将依赖上下文的问题（如 "他后来怎么样了？"）重写为独立问题（如 "爱因斯坦后来怎么样了？"），使搜索查询更加精准。

### 4.3 Researcher — 研究代理

`src/lib/agents/search/researcher/index.ts`

Researcher 是一个 **agentic 循环**，模拟人类研究行为：

```
maxIteration =
  speed: 2    |  balanced: 6  |  quality: 25
```

**循环逻辑**：

```
for each iteration:
  1. 根据 mode 构建 researcher prompt（含可用工具描述）
  2. LLM streamText (带 tools)
  3. 收集 tool calls（支持流式增量解析）
  4. 如果调用 done → break
  5. 将 tool results 加入 message history
  6. ActionRegistry.executeAll() 并行执行所有工具
```

**`__reasoning_preamble` 机制**：balanced 和 quality 模式下，LLM 必须在每次工具调用前先调用一个特殊的 `__reasoning_preamble` 工具来陈述推理过程。这个推理被实时推送到前端的 research block 中，实现可视化的思考过程。

### 4.4 ActionRegistry — 研究工具注册表

`src/lib/agents/search/researcher/actions/registry.ts`

采用注册表模式，所有研究工具统一管理：

```
已注册的 7 个工具：
├── web_search        → 网页搜索（SearxNG + 嵌入去重）
├── academic_search   → 学术搜索（arxiv + Google Scholar + PubMed）
├── social_search     → 社区讨论搜索（Reddit 等）
├── uploads_search    → 上传文件语义搜索
├── scrape_url        → 抓取指定 URL
├── plan              → 规划下一步行动
└── done              → 结束研究循环
```

每个工具实现 `ResearchAction` 接口：

```typescript
interface ResearchAction {
  name: string;
  schema: z.ZodObject<any>;            // Zod 输入验证
  getToolDescription(config): string;   // LLM 可见的工具描述
  getDescription(config): string;       // prompt 中的详细说明
  enabled(config): boolean;             // 是否在此上下文中启用
  execute(params, config): Promise<ActionOutput>;
}
```

**动态启用逻辑**：工具的 `enabled` 方法根据分类结果决定是否可用。例如 `academic_search` 只在 `classification.academicSearch === true && sources.includes('academic')` 时启用。

### 4.5 WidgetExecutor — Widget 系统

`src/lib/agents/search/widgets/executor.ts`

Widgets 是独立于搜索的即时信息卡片：

```
已注册的 3 个 Widget：
├── weatherWidget     → 天气 (Open-Meteo API + Nominatim 地理编码)
├── stockWidget       → 股票 (Yahoo Finance API)
└── calculationWidget → 计算器 (mathjs 表达式求值)
```

**执行流程**：

1. `WidgetExecutor.executeAll()` 遍历所有已注册 Widget
2. 每个 Widget 的 `shouldExecute(classification)` 判断是否需要执行
3. 并行执行所有匹配的 Widget
4. 每个.Widget 内部先用 LLM `generateObject` 提取参数（如地点名、股票代码），再调用外部 API

### 4.6 SessionManager — 会话管理

`src/lib/session.ts`

管理每个请求的生命周期，核心功能：

- **EventEmitter 模式**：`emit('data'|'end'|'error')` + `subscribe(listener)`
- **Block 存储**：`Map<string, Block>` 存储当前会话的所有 Block
- **RFC 6902 JSON Patch**：`updateBlock()` 使用标准 JSON Patch 协议增量更新 Block
- **事件回放**：新订阅者会收到历史事件（支持断线重连）
- **TTL 清理**：30 分钟后自动清理会话

```typescript
// 关键：使用 rfc6902 库执行 JSON Patch
updateBlock(blockId, patch) {
  applyPatch(block, patch);  // RFC 6902
  this.emit('data', { type: 'updateBlock', blockId, patch });
}
```

---

## 5. 搜索流水线深度解析

### 5.1 基础搜索 executeSearch

`src/lib/agents/search/researcher/actions/search/baseSearch.ts`

这是所有搜索类型的核心函数，根据模式分两条路径：

#### Speed / Balanced 模式（嵌入去重路径）

```
1. 对每个 query 调用 searchSearxng()
2. 对每个结果计算 query 与内容的嵌入相似度
3. 过滤 similarity > 0.5 的结果
4. 嵌入空间去重：结果间相似度 > 0.75 视为重复
5. 按相似度排序，取 Top 20
```

#### Quality 模式（深度研究路径）

```
1. 搜索 → 获取所有结果（不做嵌入过滤）
2. LLM Picker：让 LLM 从结果中挑选 2-3 个最相关的
3. Playwright 抓取选中页面的完整内容
4. 文本分块（4000 tokens/chunk, 500 overlap）
5. LLM Extractor：从每个 chunk 中提取关键事实
6. 返回精炼的事实摘要
```

**Quality 模式的 LLM 调用链**：

```
SearxNG 搜索 → LLM Picker (选结果)
            → Playwright 抓取 (获取全文)
            → LLM Extractor × N chunks (逐块提取事实)
```

这意味着 quality 模式会产生大量 LLM 调用（每个搜索结果 × 每个 chunk），但提供了最深度的信息提取。

### 5.2 SearxNG 集成

`src/lib/searxng.ts`

```typescript
// 简单的 HTTP 调用，10s 超时
const searchSearxng = async (query, opts?) => {
  const url = new URL(`${searxngURL}/search?format=json`);
  url.searchParams.append('q', query);
  // opts: categories, engines, language, pageno
  const res = await fetch(url, { signal: controller.signal });
  return { results, suggestions };
};
```

**搜索类型配置**：
- `webSearch`：默认配置
- `academicSearch`：`{ engines: ['arxiv', 'google scholar', 'pubmed'] }`
- `socialSearch`：社区讨论类别

### 5.3 网页抓取 Scraper

`src/lib/scraper.ts`

使用 **Playwright headless Chromium** 抓取网页：

```
关键设计：
├── 单例浏览器 + 引用计数
├── Mutex 保证并发安全
├── 30 秒空闲后自动关闭浏览器（节省内存）
├── 反自动化检测：隐藏 webdriver 属性
├── 自定义 User-Agent
├── Readability (Mozilla) 提取正文内容
└── JSDOM 解析 HTML
```

### 5.4 文件上传与语义搜索

`src/lib/uploads/manager.ts` + `src/lib/uploads/store.ts`

**上传流程**：

```
文件上传 → 内容提取 → 文本分块(512/128) → 嵌入向量化 → 存储 .content.json
         ├── text/plain:  直接读取
         ├── PDF:         pdf-parse 提取文本
         └── Word(.docx): officeparser 提取文本
```

**语义搜索**（UploadStore.query）：

```
1. 对查询进行嵌入
2. 与所有 chunk 的嵌入计算余弦相似度
3. Reciprocal Rank Fusion (RRF) 融合多查询结果
   score = Σ similarity / (rank + 60)
4. 返回 Top-K 结果
```

---

## 6. 多 LLM 提供商系统

### 6.1 提供商架构

Vane 设计了一个优雅的三层抽象：

```
ModelRegistry (注册表)
  └── BaseModelProvider<CONFIG> (抽象基类)
       ├── getDefaultModels()
       ├── getModelList()
       ├── loadChatModel(modelName) → BaseLLM
       └── loadEmbeddingModel(modelName) → BaseEmbedding
              │                          │
              ▼                          ▼
         BaseLLM<CONFIG>           BaseEmbedding<CONFIG>
         ├── generateText()        ├── embedText()
         ├── streamText()          └── embedTextBatch()
         ├── generateObject()
         └── streamObject()
```

### 6.2 已支持的 8 个提供商

| Provider | 类型 | LLM | Embedding | 说明 |
|----------|------|-----|-----------|------|
| `openai` | 云端 | ✅ | ✅ | GPT 系列 + text-embedding |
| `anthropic` | 云端 | ✅ | ❌ | Claude 系列 |
| `gemini` | 云端 | ✅ | ✅ | Google Gemini |
| `groq` | 云端 | ✅ | ❌ | 高速推理 |
| `ollama` | 本地 | ✅ | ✅ | 本地 LLM 运行时 |
| `lmstudio` | 本地 | ✅ | ✅ | 本地模型管理 |
| `lemonade` | 本地 | ✅ | ✅ | AMD GPU 加速 |
| `transformers` | 本地 | ❌ | ✅ | HuggingFace Transformers.js |

### 6.3 配置系统

`src/lib/config/index.ts`

Vane 使用 **JSON 文件配置**（`data/config.json`），支持运行时通过 UI 修改：

```typescript
type Config = {
  version: number;
  setupComplete: boolean;
  preferences: { theme, measureUnit, autoMediaSearch, ... };
  personalization: { systemInstructions };
  modelProviders: ConfigModelProvider[];
  search: { searxngURL };
};
```

**环境变量自动发现**：每个 Provider 声明其所需的 env vars 和 config fields，ConfigManager 在启动时自动扫描环境变量，创建配置好的 Provider 实例。

**首次设置向导**：`setupComplete` 标志控制是否显示设置界面。用户通过 UI 配置 API Key、模型选择、SearxNG URL 等。

---

## 7. 流式通信协议

### 7.1 NDJSON 流

Vane 使用 **Newline-Delimited JSON (NDJSON)** 作为流式传输格式：

```
data: {"type":"block","block":{"id":"...","type":"research","data":{"subSteps":[]}}}
data: {"type":"updateBlock","blockId":"...","patch":[{"op":"replace","path":"/data/subSteps","value":[...]}]}
data: {"type":"updateBlock","blockId":"...","patch":[{"op":"replace","path":"/data","value":"partial answer..."}]}
data: {"type":"researchComplete"}
data: {"type":"messageEnd"}
```

### 7.2 三种消息类型

```
block        → 新增一个 Block（如开始研究、开始回答）
updateBlock  → 增量更新 Block（RFC 6902 JSON Patch）
researchComplete → 研究阶段完成标记
messageEnd   → 整个回答完成
```

### 7.3 RFC 6902 JSON Patch

```typescript
// 首个 chunk → 创建新 Block
session.emitBlock({ id, type: 'text', data: chunk });

// 后续 chunk → 增量更新（使用 patch）
session.updateBlock(blockId, [{
  op: 'replace',
  path: '/data',
  value: accumulatedText  // 完整替换文本
}]);
```

这种设计让前端可以精确知道哪个部分发生了变化，实现高效的 DOM 更新。

---

## 8. 前端架构

### 8.1 技术选型

- **Next.js 16 App Router**：React Server Components + Client Components
- **TailwindCSS + Headless UI**：无样式可访问组件
- **Phosphor Icons**：图标系统
- **react-syntax-highlighter**：代码高亮
- **markdown-to-jsx**：Markdown 渲染
- **jspdf**：导出 PDF
- **lightweight-charts**：股票图表
- **yet-another-react-lightbox**：图片灯箱

### 8.2 核心组件

Vane 的前端（在 `morphic/` 子模块中）包含丰富的组件：

```
components/
├── artifact/              → 研究过程 + 答案展示
│   ├── artifact-root.tsx
│   ├── search-artifact-content.tsx
│   ├── reasoning-content.tsx
│   ├── tool-invocation-content.tsx
│   └── todo-invocation-content.tsx
├── chat-panel.tsx         → 主聊天面板
├── answer-section.tsx     → 答案区域
├── search-mode-selector.tsx → 模式选择器
├── app-sidebar.tsx        → 侧边栏（历史记录）
├── attachment-preview.tsx → 文件预览
└── auth-modal.tsx         → 认证弹窗
```

### 8.3 流式数据消费

前端通过 `fetch` + `ReadableStream` 消费 NDJSON 流，逐步渲染：
- Block 创建 → 渲染新区域
- Block 更新 → 应用 JSON Patch
- Research subSteps → 动态追加研究过程可视化

---

## 9. API 接口设计

### 9.1 API 端点概览

| 方法 | 路径 | 功能 |
|------|------|------|
| POST | `/api/chat` | 聊天接口（SSE 流式） |
| POST | `/api/search` | 搜索 API（支持流式/非流式） |
| POST | `/api/images` | 图片搜索 |
| POST | `/api/videos` | 视频搜索 |
| POST | `/api/uploads` | 文件上传 |
| GET | `/api/chats` | 获取聊天列表 |
| GET | `/api/chats/[id]` | 获取特定聊天 |
| GET | `/api/providers` | 列出已配置的 Provider |
| POST | `/api/providers` | 添加 Provider |
| PUT | `/api/providers/[id]` | 更新 Provider |
| DELETE | `/api/providers/[id]` | 删除 Provider |
| GET | `/api/providers/[id]/models` | 获取 Provider 的模型列表 |
| GET | `/api/config` | 获取配置 |
| POST | `/api/config/setup-complete` | 标记设置完成 |
| POST | `/api/suggestions` | 搜索建议 |
| GET | `/api/discover` | 发现页内容 |
| GET | `/api/weather` | 天气数据 |
| POST | `/api/reconnect/[id]` | 重连会话 |

### 9.2 Chat API 详解

`POST /api/chat`

**请求体**：

```typescript
{
  message: { messageId, chatId, content },  // 消息内容
  optimizationMode: 'speed' | 'balanced' | 'quality',
  sources: SearchSources[],                 // ['web', 'academic', ...]
  history: [['human', '...'], ['assistant', '...']],  // 聊天历史
  files: string[],                          // 上传文件 ID 列表
  chatModel: { providerId, key },           // LLM 模型选择
  embeddingModel: { providerId, key },      // 嵌入模型选择
  systemInstructions: string                // 用户自定义指令
}
```

**响应**：`text/event-stream`，NDJSON 格式

### 9.3 Search API 详解

`POST /api/search`

程序化搜索接口，使用 `APISearchAgent`（简化版的 SearchAgent）：

```typescript
// 非流式模式
{ message: string, sources: Chunk[] }

// 流式模式 (stream: true)
data: {"type":"init","data":"Stream connected"}
data: {"type":"sources","data":[...]}
data: {"type":"response","data":"partial..."}
data: {"type":"done"}
```

---

## 10. 部署与配置

### 10.1 三种部署方式

**① Docker 一键部署（推荐）**：

```bash
# 内置 SearxNG
docker run -d -p 3000:3000 -v vane-data:/home/vane/data \
  --name vane itzcrazykns1337/vane:latest
```

**② Docker Slim（自带 SearxNG）**：

```bash
docker run -d -p 3000:3000 \
  -e SEARXNG_API_URL=http://your-searxng:8080 \
  -v vane-data:/home/vane/data \
  --name vane itzcrazykns1337/vane:slim-latest
```

**③ 非 Docker 部署**：

```bash
npm i && npm run build && npm run start
# 需自行安装 SearxNG 并启用 JSON + Wolfram Alpha
```

### 10.2 Dockerfile 设计

Vane 提供两个 Dockerfile：

- `Dockerfile`：完整版（Vane + SearxNG），多阶段构建
- `Dockerfile.slim`：精简版（仅 Vane，外部 SearxNG）

`entrypoint.sh` 负责启动 SearxNG 和 Vane 两个服务。

### 10.3 SearxNG 配置要求

```
必须配置：
├── 启用 JSON format 输出
└── 启用 Wolfram Alpha 搜索引擎

settings.yml 示例:
search:
  formats:
    - html
    - json
```

### 10.4 数据持久化

```
data/
├── config.json              → 全局配置
├── vane.db                  → SQLite 数据库
└── uploads/                 → 上传文件
    ├── uploaded_files.json  → 文件记录
    ├── <hash>.pdf           → 原始文件
    └── <hash>.content.json  → 分块+嵌入数据
```

---

## 11. 架构亮点与改进方向

### 11.1 架构亮点

**① 注册表模式贯穿全局**

ActionRegistry（研究工具）、WidgetExecutor（Widgets）、providers（LLM 提供商）均采用注册表模式。新增工具只需实现接口并注册，无需修改核心逻辑。**扩展性极强**。

**② 模式驱动的 Prompt 工程**

speed / balanced / quality 三种模式不仅影响迭代次数（2/6/25），还完全改变 Prompt 内容、搜索策略（嵌入去重 vs 深度抓取）和答案生成深度。**同一套代码，三种体验**。

**③ RFC 6902 JSON Patch 流式协议**

使用标准化的 JSON Patch 协议进行 Block 增量更新，而非简单的文本拼接。这让前端可以精确控制 UI 更新范围。**工程化程度高**。

**④ 上下文感知的分类器**

standaloneFollowUp 机制将依赖上下文的追问重写为独立问题，极大提升了多轮对话中的搜索质量。**贴近真实搜索场景**。

**⑤ Provider 完全解耦**

每个 LLM Provider 是独立的类，通过统一的 BaseLLM / BaseEmbedding 接口与系统集成。新增 Provider 只需实现 4 个方法。**符合开闭原则**。

**⑥ Quality 模式的深度研究链**

LLM Picker → Playwright 抓取 → LLM Extractor 的三级流水线，模拟人类 "搜索 → 阅读 → 提炼" 的研究过程。**信息质量显著高于传统搜索**。

### 11.2 潜在改进方向

**① 缺少向量数据库**

当前文件搜索使用内存中的线性扫描 + 余弦相似度，对于大量上传文件会有效率问题。可集成 SQLite-vss 或 ChromaDB。

**② Quality 模式的 LLM 调用成本**

每个搜索结果 × 每个 chunk 都会调用一次 LLM Extractor。在 quality 模式下可能产生数十次 LLM 调用，成本较高。

**③ 无认证系统**

当前版本完全无认证，README 中将 "Adding authentication" 列为 upcoming feature。公网部署存在安全风险。

**④ SessionManager 内存管理**

会话存储在内存 Map 中，30 分钟 TTL。高并发场景下可能导致内存压力。可考虑 Redis 外部化。

**⑤ 嵌入模型依赖**

speed/balanced 模式强依赖嵌入模型进行去重。如果嵌入模型不可用（如只配置了 Anthropic），会 fallback 到无去重模式，结果质量下降。

**⑥ morphic 子模块耦合**

前端深度依赖 morphic 子模块，两个项目的关系不够清晰（fork? 子项目? 引用?），增加了理解和维护成本。

---

## 12. 关键文件索引

### 核心编排

| 文件 | 行数 | 说明 |
|------|------|------|
| `src/lib/agents/search/index.ts` | 192 | SearchAgent 主编排器 |
| `src/lib/agents/search/classifier.ts` | 53 | 分类器 |
| `src/lib/agents/search/types.ts` | 123 | 核心类型定义 |
| `src/lib/agents/search/researcher/index.ts` | 223 | Researcher agentic 循环 |
| `src/lib/agents/search/researcher/actions/registry.ts` | 107 | 工具注册表 |
| `src/lib/agents/search/researcher/actions/index.ts` | 18 | 工具注册入口 |
| `src/lib/agents/search/researcher/actions/search/baseSearch.ts` | 423 | 基础搜索（核心） |
| `src/lib/agents/search/researcher/actions/search/webSearch.ts` | 114 | 网页搜索 |
| `src/lib/agents/search/researcher/actions/search/academicSearch.ts` | 62 | 学术搜索 |
| `src/lib/agents/search/widgets/executor.ts` | 36 | Widget 执行器 |

### Prompt 工程

| 文件 | 行数 | 说明 |
|------|------|------|
| `src/lib/prompts/search/classifier.ts` | 64 | 分类器 Prompt |
| `src/lib/prompts/search/researcher.ts` | 354 | 研究员 Prompt（三模式） |
| `src/lib/prompts/search/writer.ts` | 54 | 答案生成 Prompt |

### LLM 系统

| 文件 | 行数 | 说明 |
|------|------|------|
| `src/lib/models/base/llm.ts` | 22 | BaseLLM 抽象 |
| `src/lib/models/base/embedding.ts` | - | BaseEmbedding 抽象 |
| `src/lib/models/base/provider.ts` | 45 | BaseModelProvider 抽象 |
| `src/lib/models/registry.ts` | 221 | ModelRegistry 注册表 |
| `src/lib/models/providers/index.ts` | 35 | 8 个 Provider 注册 |

### 搜索与抓取

| 文件 | 行数 | 说明 |
|------|------|------|
| `src/lib/searxng.ts` | 67 | SearxNG HTTP 封装 |
| `src/lib/scraper.ts` | 116 | Playwright 网页抓取 |

### 存储与配置

| 文件 | 行数 | 说明 |
|------|------|------|
| `src/lib/db/schema.ts` | 38 | SQLite Schema（2 表） |
| `src/lib/config/index.ts` | 390 | ConfigManager 配置管理 |
| `src/lib/config/types.ts` | 109 | 配置类型定义 |
| `src/lib/uploads/manager.ts` | 217 | 文件上传管理 |
| `src/lib/uploads/store.ts` | 120 | 文件语义搜索 |

### API 路由

| 文件 | 说明 |
|------|------|
| `src/app/api/chat/route.ts` | 聊天 API（SSE） |
| `src/app/api/search/route.ts` | 搜索 API（流式/非流式） |
| `src/app/api/uploads/route.ts` | 文件上传 |
| `src/app/api/providers/route.ts` | Provider CRUD |
| `src/app/api/chats/route.ts` | 聊天历史 |

### 基础设施

| 文件 | 行数 | 说明 |
|------|------|------|
| `src/lib/session.ts` | 105 | 会话管理 + EventEmitter |
| `src/lib/types.ts` | 123 | 全局类型（Block, Chunk, Message） |
| `src/lib/utils/splitText.ts` | - | 文本分块工具 |
| `src/lib/utils/computeSimilarity.ts` | - | 余弦相似度 |
| `src/lib/utils/formatHistory.ts` | - | 聊天历史格式化 |

---

## 附录：完整请求时序图

```
用户                  Vane API              Classifier    Researcher    Widgets    Writer LLM    SearxNG
 │                       │                      │            │           │            │           │
 │── POST /api/chat ────▶│                      │            │           │            │           │
 │                       │── create session ────│            │           │            │           │
 │                       │── classify() ───────▶│            │           │            │           │
 │                       │◀── classification ───│            │           │            │           │
 │                       │                      │            │           │            │           │
 │◀── SSE stream start ──│                      │            │           │            │           │
 │                       │── Promise.all ───────┼───────────▶│           │            │           │
 │                       │                      │            │           │            │           │
 │                       │                      │            │           │── execute ─│           │
 │                       │                      │            │           │◀─ results ─│           │
 │◀── widget block ──────│                      │            │           │            │           │
 │                       │                      │            │── search ─┼────────────┼──────────▶│
 │◀── research block ────│                      │            │◀─ results ┼────────────┼───────────│
 │    (searching...)     │                      │            │           │            │           │
 │◀── research update ───│                      │            │── scrape ─┼────────────┼──────────▶│
 │    (reading...)       │                      │            │◀─ content ┼────────────┼───────────│
 │                       │                      │            │── done ───│            │           │
 │◀── research update ───│                      │            │           │            │           │
 │                       │◀── searchFindings ───│            │           │            │           │
 │                       │                      │            │           │            │           │
 │◀── researchComplete ──│                      │            │           │            │           │
 │                       │── streamText ────────┼────────────┼───────────┼───────────▶│           │
 │◀── text block ────────│                      │            │           │            │           │
 │◀── text update ───────│  (incremental patch) │            │           │            │           │
 │◀── text update ───────│                      │            │           │            │           │
 │◀── messageEnd ────────│                      │            │           │            │           │
 │                       │── save to DB ────────│            │           │            │           │
```

---

*文档生成时间: 2026-07-30 | 基于 Vane v1.12.2 源码分析*

---

☕️ 制作不易，请我喝咖啡☕️关注我➕