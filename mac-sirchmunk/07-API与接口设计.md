# 7. API 与接口设计（API & Interface Design）

> 本章详细描述 Sirchmunk 的所有对外及内部 API，包括请求/响应示例、参数说明、认证、限流、版本控制等。

---

## 7.0 API 设计原则与版本策略

### 7.0.1 设计原则

| 原则 | 说明 |
|------|------|
| RESTful | 资源导向的 URL 设计 |
| 版本化 | URL 路径版本控制（`/api/v1/`） |
| 统一响应 | `{success, data, error}` 标准结构 |
| 流式优先 | 实时数据使用 WebSocket |
| 安全默认 | Token 认证 + CORS + 安全头 |

### 7.0.2 版本策略

- **当前版本**：`v1`
- **版本位置**：URL 路径（`/api/v1/...`）
- **兼容性**：同一版本内向后兼容
- **弃用策略**：旧版本保留至少一个版本周期

---

## 7.1 API 全景列表

### 7.1.1 REST API 端点

| 方法 | 路由 | 模块 | 功能 |
|------|------|------|------|
| GET | `/` | main | API 信息（无 UI 时） |
| GET | `/health` | main | 健康检查 |
| POST | `/api/v1/search` | search | 执行搜索 |
| GET | `/api/v1/knowledge/list` | knowledge | 知识簇列表 |
| GET | `/api/v1/knowledge/{id}` | knowledge | 知识簇详情 |
| DELETE | `/api/v1/knowledge/{id}` | knowledge | 删除知识簇 |
| POST | `/api/v1/files/upload` | files | 文件上传 |
| POST | `/api/v1/files/check-duplicates` | files | 重复检测 |
| GET | `/api/v1/files/collections` | files | 集合列表 |
| GET | `/api/v1/chat/sessions` | history | 会话列表 |
| GET | `/api/v1/chat/sessions/{id}` | history | 会话详情 |
| DELETE | `/api/v1/chat/sessions/{id}` | history | 删除会话 |
| GET | `/api/v1/history/search` | history | 搜索历史 |
| GET | `/api/v1/history/stats` | history | 历史统计 |
| GET | `/api/v1/dashboard/recent` | history | 最近活动 |
| GET | `/api/v1/monitor/overview` | monitor | 监控概览 |
| GET | `/api/v1/monitor/system` | monitor | 系统指标 |
| GET | `/api/v1/monitor/health` | monitor | 健康状态 |
| GET | `/api/v1/monitor/chat` | monitor | 聊天活动 |
| GET | `/api/v1/settings/ui` | settings | UI 设置 |
| POST | `/api/v1/settings/ui` | settings | 更新 UI 设置 |
| GET | `/api/v1/settings/env` | settings | 环境变量 |
| POST | `/api/v1/settings/env` | settings | 更新环境变量 |
| GET | `/api/v1/tools` | tools | 工具列表 |
| POST | `/api/v1/tools/{id}` | tools | 执行工具 |

### 7.1.2 WebSocket 端点

| 路由 | 模块 | 功能 |
|------|------|------|
| `/api/v1/chat` | chat | 实时聊天 |

### 7.1.3 MCP Tools

| 工具名 | 模块 | 功能 |
|--------|------|------|
| `sirchmunk_search` | mcp/tools | 搜索本地文件 |
| `sirchat` | mcp/tools | 聊天交互 |

---

## 7.2 Search API

### 7.2.1 `POST /api/v1/search`

**功能**：执行 Agentic Search 搜索。

**请求头**：
```
Authorization: Bearer <token>
Content-Type: application/json
```

**请求体**：
```json
{
    "query": "什么是 QLoRA？",
    "mode": "FAST",
    "max_loops": 10,
    "max_token_budget": 64000,
    "include": ["*.py", "*.md"],
    "exclude": ["node_modules/**", ".git/**"],
    "return_context": false
}
```

**参数说明**：

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `query` | string | 是 | — | 搜索查询文本 |
| `mode` | string | 否 | `FAST` | 搜索模式（FAST/DEEP） |
| `max_loops` | int | 否 | 10 | 最大 ReAct 循环次数 |
| `max_token_budget` | int | 否 | 64000 | LLM token 预算 |
| `include` | string[] | 否 | — | 文件包含模式（glob） |
| `exclude` | string[] | 否 | — | 文件排除模式（glob） |
| `return_context` | bool | 否 | false | 是否返回 SearchContext |

**成功响应**（200）：
```json
{
    "success": true,
    "data": {
        "answer": "QLoRA 是一种参数高效微调方法...",
        "mode": "FAST",
        "duration_ms": 12500
    }
}
```

**带上下文的成功响应**（`return_context=true`）：
```json
{
    "success": true,
    "data": {
        "answer": "QLoRA 是一种参数高效微调方法...",
        "context": {
            "max_token_budget": 64000,
            "total_llm_tokens": 12500,
            "loop_count": 5,
            "read_file_ids": ["/path/to/file1.pdf", "/path/to/file2.md"],
            "retrieval_logs": [
                {
                    "tool_name": "keyword_search",
                    "tokens": 0,
                    "metadata": {"keywords": ["QLoRA"]}
                }
            ],
            "search_history": ["QLoRA", "量化低秩适配"]
        }
    }
}
```

**错误响应**（400/500）：
```json
{
    "success": false,
    "error": {
        "code": "INVALID_REQUEST",
        "message": "Query cannot be empty",
        "details": "..."
    }
}
```

---

## 7.3 Chat API（WebSocket）

### 7.3.1 `WebSocket /api/v1/chat`

**功能**：实时双向聊天通信。

**连接 URL**：
```
ws://localhost:8584/api/v1/chat
wss://example.com/api/v1/chat?token=<bearer_token>
```

**认证方式**：
- Query 参数：`?token=<bearer_token>`
- Header：`Authorization: Bearer <token>`

### 7.3.2 客户端 → 服务端消息

**发送聊天消息**：
```json
{
    "type": "chat",
    "message": "什么是 QLoRA？",
    "session_id": "uuid-session-123",
    "mode": "FAST",
    "enable_rag": true,
    "enable_web_search": false,
    "search_mode": "hybrid"
}
```

**停止生成**：
```json
{
    "type": "stop",
    "session_id": "uuid-session-123"
}
```

**心跳**：
```json
{
    "type": "ping"
}
```

### 7.3.3 服务端 → 客户端消息

**Token 流式输出**：
```json
{
    "type": "token",
    "content": "QLoRA",
    "session_id": "uuid-session-123"
}
```

**阶段更新**：
```json
{
    "type": "stage",
    "stage": "searching",
    "message": "正在搜索相关文档..."
}
```

**搜索结果**：
```json
{
    "type": "search_result",
    "files": ["/path/to/file1.pdf", "/path/to/file2.md"]
}
```

**完成**：
```json
{
    "type": "done",
    "session_id": "uuid-session-123",
    "duration_ms": 12500
}
```

**错误**：
```json
{
    "type": "error",
    "message": "LLM 调用失败",
    "code": "LLM_ERROR"
}
```

**心跳响应**：
```json
{
    "type": "pong"
}
```

---

## 7.4 Knowledge API

### 7.4.1 `GET /api/v1/knowledge/list`

**功能**：列出所有知识簇。

**请求参数**：

| 参数 | 类型 | 说明 |
|------|------|------|
| `limit` | int | 返回数量（默认 20） |
| `offset` | int | 偏移量（默认 0） |
| `lifecycle` | string | 按生命周期过滤 |

**成功响应**：
```json
{
    "success": true,
    "data": [
        {
            "id": "C1001",
            "name": "QLoRA: 4-bit Quantized Low-Rank Adaptation",
            "confidence": 0.92,
            "lifecycle": "stable",
            "evidences_count": 5,
            "last_modified": "2024-01-01T00:00:00Z"
        }
    ],
    "pagination": {
        "limit": 20,
        "offset": 0,
        "total": 15
    }
}
```

### 7.4.2 `GET /api/v1/knowledge/{cluster_id}`

**功能**：获取知识簇详情。

**成功响应**：
```json
{
    "success": true,
    "data": {
        "id": "C1001",
        "name": "QLoRA: 4-bit Quantized Low-Rank Adaptation",
        "description": "QLoRA 是一种...",
        "content": "# QLoRA\n\n## 核心思想...",
        "evidences": [
            {
                "doc_id": "abc123",
                "file_or_url": "/path/to/paper.pdf",
                "summary": "QLoRA 通过 4-bit 量化...",
                "is_found": true,
                "snippets": [...]
            }
        ],
        "patterns": ["低秩分解", "量化压缩"],
        "constraints": [
            {
                "condition": "gpu_memory < 24GB",
                "severity": "medium",
                "description": "需要足够的 GPU 内存"
            }
        ],
        "confidence": 0.92,
        "lifecycle": "stable",
        "version": 3
    }
}
```

### 7.4.3 `DELETE /api/v1/knowledge/{cluster_id}`

**功能**：删除知识簇。

**成功响应**：
```json
{
    "success": true,
    "message": "Knowledge cluster C1001 deleted"
}
```

---

## 7.5 Files API

### 7.5.1 `POST /api/v1/files/upload`

**功能**：批量上传文件到指定集合。

**请求头**：
```
Content-Type: multipart/form-data
Authorization: Bearer <token>
```

**请求参数**（form-data）：

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `files` | File[] | 是 | 上传的文件列表 |
| `collection` | string | 否 | 集合名称（默认 "default"） |
| `paths` | string[] | 否 | 相对路径（保留目录结构） |
| `overwrite` | boolean | 否 | 是否覆盖（默认 false） |

**成功响应**：
```json
{
    "success": true,
    "data": {
        "uploaded": [
            {
                "name": "paper.pdf",
                "size": 1048576,
                "sha256": "a1b2c3d4..."
            }
        ],
        "skipped": [
            {
                "name": "duplicate.pdf",
                "reason": "already_exists"
            }
        ],
        "collection": "default"
    }
}
```

### 7.5.2 `POST /api/v1/files/check-duplicates`

**功能**：检查文件是否已存在于集合中。

**请求体**：
```json
{
    "collection": "default",
    "files": [
        {"name": "paper.pdf", "size": 1048576},
        {"name": "new.docx", "size": 524288}
    ]
}
```

**成功响应**：
```json
{
    "success": true,
    "data": {
        "duplicates": [
            {"name": "paper.pdf", "size": 1048576}
        ],
        "new_files": [
            {"name": "new.docx", "size": 524288}
        ]
    }
}
```

---

## 7.6 History API

### 7.6.1 `GET /api/v1/chat/sessions`

**功能**：获取聊天会话列表。

**请求参数**：

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `limit` | int | 20 | 返回数量 |
| `offset` | int | 0 | 偏移量 |

**成功响应**：
```json
{
    "success": true,
    "data": [
        {
            "session_id": "uuid-123",
            "title": "什么是 QLoRA？",
            "message_count": 4,
            "last_message": "QLoRA 是一种参数高效...",
            "created_at": 1704067200,
            "updated_at": 1704067300,
            "topics": []
        }
    ],
    "pagination": {
        "limit": 20,
        "offset": 0,
        "total": 5
    }
}
```

### 7.6.2 `GET /api/v1/chat/sessions/{session_id}`

**功能**：获取会话详情。

**成功响应**：
```json
{
    "success": true,
    "data": {
        "session_id": "uuid-123",
        "title": "什么是 QLoRA？",
        "messages": [
            {
                "role": "user",
                "content": "什么是 QLoRA？",
                "timestamp": 1704067200
            },
            {
                "role": "assistant",
                "content": "QLoRA 是一种...",
                "timestamp": 1704067250,
                "search_logs": [...]
            }
        ],
        "settings": {"mode": "FAST"},
        "created_at": 1704067200,
        "updated_at": 1704067300
    }
}
```

### 7.6.3 `DELETE /api/v1/chat/sessions/{session_id}`

**功能**：删除会话。

**成功响应**：
```json
{
    "success": true,
    "message": "Chat session deleted successfully",
    "data": {"session_id": "uuid-123"}
}
```

### 7.6.4 `GET /api/v1/history/search`

**功能**：搜索聊天历史。

**请求参数**：

| 参数 | 类型 | 说明 |
|------|------|------|
| `query` | string | 搜索关键词 |
| `limit` | int | 返回数量（默认 20） |

### 7.6.5 `GET /api/v1/history/stats`

**功能**：获取历史统计信息。

**成功响应**：
```json
{
    "success": true,
    "data": {
        "total_sessions": 15,
        "total_messages": 120,
        "recent_activity": {
            "last_7_days": 5,
            "daily_average": 0.71
        }
    }
}
```

### 7.6.6 `GET /api/v1/dashboard/recent`

**功能**：获取最近活动。

**请求参数**：

| 参数 | 类型 | 说明 |
|------|------|------|
| `limit` | int | 返回数量（默认 50） |
| `type` | string | 过滤类型（当前仅 "chat"） |

---

## 7.7 Settings API

### 7.7.1 `GET /api/v1/settings/ui`

**功能**：获取 UI 设置。

**认证**：无需认证（白名单端点）

**成功响应**：
```json
{
    "success": true,
    "data": {
        "theme": "light",
        "language": "zh"
    }
}
```

### 7.7.2 `POST /api/v1/settings/ui`

**功能**：更新 UI 设置。

**请求体**：
```json
{
    "theme": "dark",
    "language": "en"
}
```

### 7.7.3 `GET /api/v1/settings/env`

**功能**：获取环境变量（仅暴露安全变量）。

**成功响应**：
```json
{
    "success": true,
    "data": {
        "LLM_BASE_URL": "https://api.openai.com/v1",
        "LLM_MODEL_NAME": "gpt-5.2",
        "GREP_CONCURRENT_LIMIT": "5"
    }
}
```

### 7.7.4 `POST /api/v1/settings/env`

**功能**：更新环境变量（写入 .env 文件）。

**请求体**：
```json
{
    "LLM_MODEL_NAME": "gpt-4o",
    "GREP_CONCURRENT_LIMIT": "10"
}
```

---

## 7.8 Monitor API

### 7.8.1 `GET /api/v1/monitor/overview`

**功能**：获取综合监控概览。

**成功响应**：
```json
{
    "success": true,
    "data": {
        "system": {
            "cpu_percent": 25.5,
            "memory_percent": 60.2,
            "disk_percent": 45.0,
            "uptime_seconds": 86400
        },
        "chat": {
            "total_sessions": 15,
            "total_messages": 120,
            "active_sessions": 3
        },
        "knowledge": {
            "total_clusters": 25,
            "last_24h_new": 3
        },
        "llm": {
            "total_calls": 150,
            "total_tokens": 50000,
            "calls_by_model": {
                "gpt-5.2": {"calls": 100, "tokens": 35000}
            }
        },
        "health": {
            "score": 95,
            "status": "excellent",
            "issues": []
        }
    }
}
```

### 7.8.2 `GET /api/v1/monitor/system`

**功能**：获取系统指标。

**成功响应**：
```json
{
    "success": true,
    "data": {
        "cpu": {
            "percent": 25.5,
            "count": 8,
            "freq_mhz": 3200
        },
        "memory": {
            "total_bytes": 17179869184,
            "used_bytes": 10307921510,
            "percent": 60.2
        },
        "disk": {
            "total_bytes": 500000000000,
            "used_bytes": 225000000000,
            "percent": 45.0
        },
        "network": {
            "connections": 15
        },
        "process": {
            "pid": 12345,
            "memory_mb": 512,
            "threads": 10
        }
    }
}
```

### 7.8.3 `GET /api/v1/monitor/health`

**功能**：获取健康状态。

**成功响应**：
```json
{
    "success": true,
    "data": {
        "score": 95,
        "status": "excellent",
        "issues": [],
        "services": {
            "api": "running",
            "database": "connected",
            "llm": "available",
            "embedding": "available"
        }
    }
}
```

### 7.8.4 `GET /api/v1/monitor/chat`

**功能**：获取聊天活动统计。

**请求参数**：

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `hours` | int | 24 | 时间窗口（小时） |

---

## 7.9 MCP Server Tools

### 7.9.1 `sirchmunk_search`

**功能**：搜索本地文件、文档和原始数据。

**参数**：

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `query` | string | 是 | — | 搜索查询 |
| `paths` | string[] | 否 | — | 搜索路径列表 |
| `mode` | string | 否 | `FAST` | 搜索模式 |
| `max_depth` | int | 否 | 5 | 目录深度限制 |
| `top_k_files` | int | 否 | 3 | 返回文件数 |
| `max_loops` | int | 否 | 10 | 最大循环次数 |
| `max_token_budget` | int | 否 | 128000 | Token 预算 |
| `enable_dir_scan` | bool | 否 | true | 启用目录扫描 |
| `include` | string[] | 否 | — | 包含模式 |
| `exclude` | string[] | 否 | — | 排除模式 |
| `return_context` | bool | 否 | false | 返回上下文 |

**返回**：JSON 字符串（包含 answer 和可选 context）

### 7.9.2 `sirchat`

**功能**：简化聊天接口。

**参数**：

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `message` | string | 是 | 聊天消息 |
| `history` | object[] | 否 | 历史消息 |
| `mode` | string | 否 | 搜索模式 |

**返回**：聊天响应文本

---

## 7.10 认证机制

### 7.10.1 Bearer Token 认证

**环境变量**：`SIRCHMUNK_API_TOKEN`

**请求头**：
```
Authorization: Bearer <token>
```

**验证逻辑**：
1. 如果未设置 `SIRCHMUNK_API_TOKEN`，跳过认证
2. 检查 `Authorization` 头是否以 `Bearer ` 开头
3. 使用 `hmac.compare_digest()` 比较 token（防时序攻击）

**白名单端点**（无需认证）：
- `/health`
- `/favicon.ico`
- `/_next/*`（静态资源）
- `/api/v1/settings/ui`
- `OPTIONS` 预检请求

### 7.10.2 WebSocket Token

**认证方式**（优先级）：
1. Query 参数：`?token=<token>`
2. Header：`Authorization: Bearer <token>`

### 7.10.3 Token 存储（前端）

```typescript
// 存储
localStorage.setItem("sirchmunk_api_token", token);

// 读取
const token = localStorage.getItem("sirchmunk_api_token") || "";
```

---

## 7.11 CORS / 安全头 / 限流 / 审计

### 7.11.1 CORS 配置

**环境变量**：`SIRCHMUNK_ALLOWED_ORIGINS`（逗号分隔）

```python
_allowed_origins = [o.strip() for o in _allowed_origins_raw.split(",") if o.strip()]
if not _allowed_origins:
    _allowed_origins = ["*"]  # 默认允许所有
```

**允许的方法**：`GET`, `POST`, `DELETE`, `OPTIONS`
**允许的 Headers**：`Content-Type`, `Authorization`

### 7.11.2 HTTP 安全头

| Header | 值 |
|--------|-----|
| `X-Content-Type-Options` | `nosniff` |
| `X-Frame-Options` | `DENY` |
| `Referrer-Policy` | `strict-origin-when-cross-origin` |
| `Content-Security-Policy` | `default-src 'self'; script-src 'self' 'unsafe-inline'; ...` |
| `Strict-Transport-Security` | `max-age=63072000; includeSubDomains` |
| `Permissions-Policy` | `geolocation=(), microphone=(), camera=()` |

### 7.11.3 限流（Rate Limiter）

**实现**：内存 Token Bucket

```python
class RateLimiter:
    def __init__(self, per_second: int = 5, per_minute: int = 100):
```

**限制**：
- 每秒 5 次请求
- 每分钟 100 次请求

**作用范围**：文件浏览器端点（`file_browser_limiter`）

### 7.11.4 审计日志

**格式**：JSON-Lines（每行一个 JSON 对象）

**位置**：`{work_path}/logs/audit.log`

**字段**：
```json
{
    "timestamp": "2024-01-01T00:00:00Z",
    "client_ip": "192.168.1.1",
    "action": "path_access",
    "path": "/path/to/file.pdf",
    "result": "allowed"
}
```

---

## 7.12 OpenAPI/Swagger 描述

### 7.12.1 Swagger UI

**启用条件**：`SIRCHMUNK_DEBUG=true`

**访问地址**：`http://localhost:8584/docs`

### 7.12.2 ReDoc

**访问地址**：`http://localhost:8584/redoc`

### 7.12.3 错误码规范

| HTTP 状态码 | 错误码 | 说明 |
|------------|--------|------|
| 400 | `INVALID_REQUEST` | 请求参数错误 |
| 401 | `UNAUTHORIZED` | 认证失败 |
| 403 | `FORBIDDEN` | 权限不足 |
| 404 | `NOT_FOUND` | 资源不存在 |
| 429 | `RATE_LIMITED` | 请求过于频繁 |
| 500 | `INTERNAL_ERROR` | 服务器内部错误 |

---

## 7.13 本章小结

本章完整描述了 Sirchmunk 的 API 设计：

| 类别 | 端点数 | 协议 |
|------|--------|------|
| REST API | 24 | HTTP |
| WebSocket | 1 | WS/WSS |
| MCP Tools | 2 | MCP |

**安全特性**：
- Bearer Token 认证（HMAC 比较）
- CORS 白名单
- HTTP 安全头
- 内存限流
- 审计日志
- 路径白名单 + 符号链接检测

**设计亮点**：
1. 统一响应格式（`{success, data, error}`）
2. 流式优先（WebSocket token 输出）
3. MCP 协议集成（Agent 工具暴露）
4. 版本化 URL（`/api/v1/`）

---

☕️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../res/promotion.jpg)