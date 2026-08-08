# 7. API 与接口设计（API & Interface Design）

> **文档版本**：v1.0  
> **覆盖范围**：对外 API 依赖、内部 HTTP API、MCP 协议接口、Pub/Sub 推送、认证限流

---

## 7.0 API 全景

Awesome LLM Apps 的 API 分为 3 大类：

| 类别 | 方向 | 协议 | 数量 |
|------|------|------|------|
| **外部 API 依赖** | 出站调用 | HTTPS JSON | 15+ |
| **内部 HTTP API** | 入站服务 | HTTP/REST | 5+ |
| **MCP 协议接口** | 双向 | stdio / SSE | 4+ |

---

## 7.1 外部 API 依赖

### 7.1.1 LLM 提供商 API

#### OpenAI API

| 属性 | 值 |
|------|------|
| **Base URL** | `https://api.openai.com/v1` |
| **认证** | `Authorization: Bearer <REDACTED>` |
| **主要端点** | `/chat/completions`, `/audio/speech`, `/audio/transcriptions`, `/embeddings` |
| **使用模板** | travel_agent, finance_agent_team, deep_research, voice_rag 等 |
| **限流** | Tier-based（TPM/RPM），依赖 API Key 等级 |

**请求示例**（Chat Completions）：
```http
POST /v1/chat/completions
Authorization: Bearer <REDACTED>
Content-Type: application/json

{
    "model": "gpt-4o",
    "messages": [
        {"role": "system", "content": "You are a travel planner..."},
        {"role": "user", "content": "Plan a 7-day trip to Paris"}
    ],
    "tools": [{"type": "function", "function": {"name": "search_google", ...}}],
    "temperature": 0.7,
    "max_tokens": 4096
}
```

**响应示例**：
```json
{
    "id": "chatcmpl-abc123",
    "object": "chat.completion",
    "created": 1721000000,
    "model": "gpt-4o",
    "choices": [
        {
            "index": 0,
            "message": {
                "role": "assistant",
                "content": "Here is your 7-day itinerary...",
                "tool_calls": [{"id": "call_1", "function": {"name": "search_google", "arguments": "{\"query\":\"Paris attractions\"}"}}]
            },
            "finish_reason": "stop"
        }
    ],
    "usage": {"prompt_tokens": 150, "completion_tokens": 500, "total_tokens": 650}
}
```

---

#### Google Gemini API

| 属性 | 值 |
|------|------|
| **Base URL** | `https://generativelanguage.googleapis.com/v1beta` |
| **认证** | `?key=API_KEY` 或 `Authorization: Bearer` |
| **主要端点** | `/models/{model}:generateContent`, `/models/{model}:streamGenerateContent`, `/models/{model}:embedContent` |
| **使用模板** | always_on_hn_briefing_agent, rag_chain, multimodal_ai_agent |
| **模型** | gemini-3-flash-preview, gemini-2.0-flash, gemini-1.5-pro |

**请求示例**：
```http
POST /v1beta/models/gemini-3-flash-preview:generateContent?key=API_KEY
Content-Type: application/json

{
    "contents": [{"role": "user", "parts": [{"text": "Summarize this document..."}]}],
    "generationConfig": {"temperature": 0.7, "maxOutputTokens": 2048}
}
```

---

#### Anthropic Claude API

| 属性 | 值 |
|------|------|
| **Base URL** | `https://api.anthropic.com/v1` |
| **认证** | `x-api-key: API_KEY`, `anthropic-version: 2023-06-01` |
| **主要端点** | `/messages` |
| **使用模板** | multi_mcp_agent_router, rag-as-a-service |
| **模型** | claude-sonnet-4-20250514, claude-3-5-sonnet |

**请求示例**：
```http
POST /v1/messages
x-api-key: sk-ant-xxx
anthropic-version: 2023-06-01
Content-Type: application/json

{
    "model": "claude-sonnet-4-20250514",
    "max_tokens": 4096,
    "system": "You are a code reviewer...",
    "messages": [{"role": "user", "content": "Review this code..."}],
    "tools": [{"name": "read_file", "input_schema": {...}}]
}
```

---

### 7.1.2 搜索/抓取 API

#### SerpAPI

| 属性 | 值 |
|------|------|
| **Base URL** | `https://serpapi.com/search` |
| **认证** | `?api_key=KEY` |
| **参数** | `q`（查询）, `engine`（google/google_news/google_maps）, `num`（结果数） |
| **使用模板** | travel_agent, ai_travel_agent（多个） |

**请求示例**：
```
GET https://serpapi.com/search?q=Paris+attractions&api_key=KEY&num=10
```

---

#### Firecrawl API

| 属性 | 值 |
|------|------|
| **Base URL** | `https://api.firecrawl.dev/v1` |
| **认证** | `Authorization: Bearer <REDACTED>` |
| **主要端点** | `/scrape`, `/crawl`, `/deep-research` |
| **使用模板** | deep_research_openai, ai_blog_to_podcast_agent |

**Deep Research 请求示例**：
```http
POST /v1/deep-research
Authorization: Bearer <REDACTED>

{
    "query": "Latest developments in AI agents",
    "params": {
        "maxDepth": 3,
        "timeLimit": 120,
        "maxUrls": 10
    }
}
```

---

### 7.1.3 数据 API

#### Hacker News API

| 属性 | 值 |
|------|------|
| **Base URL** | `https://hacker-news.firebaseio.com/v0` 或 `https://hn.algolia.com/api/v1` |
| **认证** | 无（公开 API） |
| **主要端点** | `/topstories.json`, `/item/{id}.json`, `/search` |
| **使用模板** | always_on_hn_briefing_agent, devpulse_ai |

**请求示例**：
```
GET https://hacker-news.firebaseio.com/v0/topstories.json
→ [12345, 12346, ...]

GET https://hacker-news.firebaseio.com/v0/item/12345.json
→ {"id": 12345, "title": "...", "url": "...", "score": 428}
```

---

#### GitHub API

| 属性 | 值 |
|------|------|
| **Base URL** | `https://api.github.com` |
| **认证** | `Authorization: token GITHUB_TOKEN`（可选，提高限流） |
| **主要端点** | `/search/repositories`, `/repos/{owner}/{repo}/issues` |
| **使用模板** | devpulse_ai, github_mcp_agent |

**请求示例**：
```
GET https://api.github.com/search/repositories?q=created:>2026-07-25+sort:stars&per_page=5
```

---

#### Yahoo Finance API（通过 yfinance 库）

| 属性 | 值 |
|------|------|
| **Base URL** | 非公开 REST（通过 `yfinance` Python 库访问） |
| **认证** | 无 |
| **使用模板** | finance_agent_team, xai_finance_agent |

---

### 7.1.4 语音 API

#### OpenAI TTS/STT

| 属性 | 值 |
|------|------|
| **Base URL** | `https://api.openai.com/v1` |
| **TTS 端点** | `/audio/speech` |
| **STT 端点** | `/audio/transcriptions` |
| **使用模板** | voice_rag_openaisdk, ai_audio_tour_agent |

---

### 7.1.5 投递 API

#### Gmail API

| 属性 | 值 |
|------|------|
| **Base URL** | `https://gmail.googleapis.com/gmail/v1` |
| **认证** | OAuth2 `Authorization: Bearer <REDACTED>` |
| **主要端点** | `/users/me/messages/send` |
| **使用模板** | always_on_hn_briefing_agent |

**请求示例**：
```http
POST /gmail/v1/users/me/messages/send
Authorization: Bearer <REDACTED>
Content-Type: application/json

{
    "raw": "RnJvbTog...base64..."
}
```

---

#### Webhook

| 属性 | 值 |
|------|------|
| **URL** | `AGENTSCOUT_WEBHOOK_URL` 环境变量 |
| **认证** | 取决于目标服务 |
| **使用模板** | always_on_hn_briefing_agent |

---

## 7.2 内部 HTTP API

### 7.2.1 AgentOS FastAPI（finance_agent_team）

| 属性 | 值 |
|------|------|
| **框架** | FastAPI（通过 Agno AgentOS） |
| **启动** | `agent_os.serve(app="finance_agent_team:app")` |
| **端点** | 自动生成（基于 Team 定义） |

**推断端点**（基于 Agno AgentOS）：
```
POST /api/v1/team/run          ← 执行团队任务
GET  /api/v1/health            ← 健康检查
GET  /api/v1/agents             ← 列出 Agent
GET  /openapi.json             ← OpenAPI Schema
```

---

### 7.2.2 AgentScout Scheduler API

| 属性 | 值 |
|------|------|
| **框架** | FastAPI + Uvicorn |
| **启动** | `uvicorn scheduler_api:app --host 0.0.0.0 --port 8000` |
| **认证** | 无（需自行添加） |

#### 端点列表

| 方法 | 路径 | 功能 | 请求体 | 响应 |
|------|------|------|--------|------|
| GET | `/health` | 健康检查 | 无 | `{"status": "ok"}` |
| GET | `/agent-scout/dry-run` | 预览 Brief | Query: `top_n`, `live` | Brief JSON |
| POST | `/agent-scout/trigger` | HTTP 触发 | `{"dry_run": false, "top_n": 5}` | 执行结果 |
| POST | `/agent-scout/pubsub` | Pub/Sub 推送 | Pub/Sub envelope | 执行结果 |

#### OpenAPI 描述（YAML 风格）

```yaml
openapi: 3.0.0
info:
  title: AgentScout Scheduler API
  version: 1.0.0
  description: HTTP and Pub/Sub hooks for scheduled Hacker News briefing runs.
  
paths:
  /health:
    get:
      summary: Health check
      responses:
        '200':
          description: Service is healthy
          content:
            application/json:
              schema:
                type: object
                properties:
                  status:
                    type: string
                    example: ok

  /agent-scout/dry-run:
    get:
      summary: Preview the Hacker News brief without delivery
      parameters:
        - name: top_n
          in: query
          schema:
            type: integer
            default: 5
            minimum: 1
            maximum: 10
        - name: live
          in: query
          schema:
            type: boolean
      responses:
        '200':
          description: Rendered brief
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Brief'

  /agent-scout/trigger:
    post:
      summary: Trigger a scheduled scout run
      requestBody:
        content:
          application/json:
            schema:
              type: object
              properties:
                dry_run:
                  type: boolean
                  default: true
                live:
                  type: boolean
                top_n:
                  type: integer
                  default: 5
      responses:
        '200':
          description: Execution result

  /agent-scout/pubsub:
    post:
      summary: Cloud Pub/Sub push endpoint
      requestBody:
        content:
          application/json:
            schema:
              type: object
              properties:
                message:
                  type: object
                  properties:
                    data:
                      type: string
                      description: Base64-encoded payload
      responses:
        '200':
          description: Execution result

components:
  schemas:
    Brief:
      type: object
      properties:
        generated_at:
          type: string
          format: date-time
        watch_mode:
          type: string
          enum: [sample, live_hn]
        subject:
          type: string
        text:
          type: string
        html:
          type: string
        stories:
          type: array
          items:
            $ref: '#/components/schemas/Story'
        next_actions:
          type: array
          items:
            type: string

    Story:
      type: object
      properties:
        title:
          type: string
        url:
          type: string
        hn_url:
          type: string
        points:
          type: integer
        comments:
          type: integer
        rank:
          type: integer
        summary:
          type: string
```

#### 请求/响应示例

**POST /agent-scout/trigger**
```bash
curl -X POST http://localhost:8000/agent-scout/trigger \
  -H "Content-Type: application/json" \
  -d '{"dry_run": true, "top_n": 3, "live": false}'
```

**响应**：
```json
{
    "dry_run": true,
    "top_n": 3,
    "delivery": {
        "attempted": false,
        "sent": false,
        "status": "dry_run",
        "detail": "Set dry_run=false to use configured Gmail or webhook delivery."
    },
    "brief": {
        "generated_at": "2026-07-26T10:00:00-07:00",
        "watch_mode": "sample",
        "subject": "AgentScout Hacker News brief - 2026-07-26",
        "text": "AgentScout Hacker News Engineering Brief\n...",
        "html": "<h2>AgentScout Hacker News Engineering Brief</h2>...",
        "stories": [
            {
                "title": "Show HN: An open-source framework for reliable AI agent workflows",
                "url": "https://example.com/reliable-agent-workflows",
                "hn_url": "https://news.ycombinator.com/item?id=40100001",
                "points": 428,
                "comments": 116,
                "rank": 1,
                "summary": "Framework discussion with practical tradeoffs..."
            }
        ],
        "next_actions": [
            "Open the highest-comment thread and extract objections or implementation patterns.",
            "Promote only stories with clear engineering signal to the team digest.",
            "If this is running on a schedule, send the brief after human-readable rendering succeeds."
        ]
    }
}
```

**POST /agent-scout/pubsub**（Pub/Sub envelope）
```bash
curl -X POST http://localhost:8000/agent-scout/pubsub \
  -H "Content-Type: application/json" \
  -d '{
    "message": {
        "data": "eyJkcnlfcnVuIjogZmFsc2UsICJ0b3BfbiI6IDV9",
        "messageId": "1234567890",
        "publishTime": "2026-07-26T09:00:00Z"
    }
  }'
```

其中 `data` 是 Base64 编码的 `{"dry_run": false, "top_n": 5}`。

---

### 7.2.3 CopilotKit Runtime API（Generative UI）

| 属性 | 值 |
|------|------|
| **框架** | CopilotKit Runtime (Hono) |
| **协议** | HTTP + WebSocket/SSE |
| **使用模板** | generative-ui-starter-project, ai-dashboard-canvas-agent |

**推断端点**：
```
POST /api/copilotkit          ← Agent 执行
WS   /ws/copilotkit           ← 流式连接
```

---

## 7.3 MCP 协议接口

### 7.3.1 MCP 协议概述

Model Context Protocol (MCP) 是 Anthropic 提出的开放协议，用于 LLM 应用与外部工具/数据的标准化集成。

| 属性 | 值 |
|------|------|
| **传输方式** | stdio（本地进程）/ SSE（远程 HTTP） |
| **消息格式** | JSON-RPC 2.0 |
| **生命周期** | Initialize → Initialized → Tools/List → Tools/Call → Shutdown |

### 7.3.2 MCP 服务器配置

#### Playwright MCP（browser_mcp_agent）

```yaml
# mcp_agent.config.yaml
mcp:
  servers:
    playwright:
      command: "npx"
      args: ["@playwright/mcp@latest"]
```

**可用工具**：
| 工具 | 功能 |
|------|------|
| `navigate` | 导航到 URL |
| `click` | 点击元素 |
| `fill` | 填写表单 |
| `screenshot` | 截图 |
| `scroll` | 滚动页面 |
| `evaluate` | 执行 JavaScript |

#### GitHub MCP（multi_mcp_agent_router）

```yaml
mcp_servers:
  - {"name": "github", "command": "npx", "args": ["-y", "@modelcontextprotocol/server-github"]}
```

**可用工具**：
| 工具 | 功能 |
|------|------|
| `search_repositories` | 搜索仓库 |
| `create_issue` | 创建 Issue |
| `list_pull_requests` | 列出 PR |
| `get_file_contents` | 获取文件内容 |

#### Filesystem MCP

```yaml
mcp_servers:
  - {"name": "filesystem", "command": "npx", "args": ["-y", "@modelcontextprotocol/server-filesystem", "/tmp"]}
```

**可用工具**：
| 工具 | 功能 |
|------|------|
| `read_file` | 读取文件 |
| `write_file` | 写入文件 |
| `list_directory` | 列出目录 |

#### Fetch MCP

```yaml
mcp_servers:
  - {"name": "fetch", "command": "npx", "args": ["-y", "@modelcontextprotocol/server-fetch"]}
```

**可用工具**：
| 工具 | 功能 |
|------|------|
| `fetch` | HTTP 请求获取网页 |

---

### 7.3.3 MCP 通信示例

**Initialize 请求**：
```json
{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "initialize",
    "params": {
        "protocolVersion": "2024-11-05",
        "capabilities": {},
        "clientInfo": {"name": "agent-forge", "version": "1.0.0"}
    }
}
```

**Tools/List 请求**：
```json
{
    "jsonrpc": "2.0",
    "id": 2,
    "method": "tools/list",
    "params": {}
}
```

**Tools/Call 请求**：
```json
{
    "jsonrpc": "2.0",
    "id": 3,
    "method": "tools/call",
    "params": {
        "name": "navigate",
        "arguments": {"url": "https://github.com"}
    }
}
```

---

## 7.4 认证与安全

### 7.4.1 认证方式汇总

| API | 认证方式 | 凭证来源 |
|-----|---------|---------|
| OpenAI | Bearer <REDACTED> | 环境变量 / Streamlit 输入 |
| Gemini | API Key (query param) | 环境变量 / Streamlit 输入 |
| Anthropic | x-api-key Header | 环境变量 / Streamlit 输入 |
| SerpAPI | api_key query param | 环境变量 / Streamlit 输入 |
| Firecrawl | Bearer <REDACTED> | 环境变量 / Streamlit 输入 |
| GitHub | Token Header | 环境变量 |
| Gmail | OAuth2 Bearer | refresh_token 换 access_token |
| AgentScout Scheduler | 无 | — |
| AgentOS | 无 | — |
| MCP | 无（本地进程） | — |

### 7.4.2 API Key 管理最佳实践

```python
# 推荐：环境变量
import os
api_key = os.environ.get("OPENAI_API_KEY")

# 推荐：Streamlit secrets
api_key = st.secrets["openai_api_key"]

# 接受但需注意：Streamlit 输入框
api_key = st.text_input("API Key", type="password")

# 避免：硬编码
api_key = "sk-xxx"  # ❌ 永远不要这样做
```

### 7.4.3 限流策略

| 层级 | 策略 | 实现 |
|------|------|------|
| LLM API | 提供商限流（TPM/RPM） | 被动接受，429 时重试 |
| SerpAPI | 月度配额 | 监控使用量 |
| 内部 API | 无 | 需自行添加（slowapi） |
| MCP | 无 | 本地进程无网络限流 |

**429 处理建议**：
```python
import time
from openai import RateLimitError

try:
    response = client.chat.completions.create(...)
except RateLimitError:
    time.sleep(2 ** retry_count)  # 指数退避
```

---

## 7.5 版本控制

### 7.5.1 API 版本策略

| API | 版本方式 | 当前版本 |
|-----|---------|---------|
| OpenAI | 无（持续更新） | — |
| Gemini | v1beta | v1beta |
| Anthropic | Header: `anthropic-version: 2023-06-01` | 2023-06-01 |
| AgentScout | URL 路径（隐式） | v1 |
| MCP | protocolVersion | 2024-11-05 |

### 7.5.2 兼容性建议

1. 使用 SDK 而非直接调用 REST（自动处理版本）
2. 锁定模型 ID（如 `gpt-4o` 而非 `gpt-4`）
3. 添加 API 版本检查（MCP）

---

## 7.6 API 总结

### 7.6.1 设计优点

1. **标准化**：使用主流 REST API，文档丰富
2. **Provider-agnostic**：通过抽象层支持多提供商
3. **MCP 前瞻性**：采用开放协议，工具集成标准化

### 7.6.2 设计缺点

1. **内部 API 无认证**：scheduler_api / AgentOS 无保护
2. **无统一 API 网关**：每个模板独立暴露端点
3. **无 API 版本管理**：内部 API 无版本控制
4. **无速率限制**：内部 API 无保护

### 7.6.3 改进建议

1. 为内部 API 添加认证（API Key / OIDC）
2. 添加 API 网关（Kong / Traefik）
3. 实现速率限制（slowapi）
4. 添加 API 版本控制（`/v1/`, `/v2/`）
5. 生成 OpenAPI Schema 并发布文档

---

> **下一章**：[08-部署运维与基础设施.md](./08-部署运维与基础设施.md) — 部署架构图、Docker/K8s/CI-CD、监控日志告警。

---

☕️ 制作不易，请我喝咖啡☕️关注我➕