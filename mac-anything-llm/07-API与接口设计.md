# 第 7 章 API 与接口设计

> **章节编号**: 07  
> **预计阅读时间**: 35 分钟  
> **关联章节**: [上一章 06-数据模型与数据库设计](./06-数据模型与数据库设计.md) | [下一章 08-部署运维与基础设施](./08-部署运维与基础设施.md)

---

## 7.1 本章导读

AnythingLLM 提供 **REST API**、**WebSocket API** 和 **SSE 流式接口** 三种对外接口。所有 REST API 挂载在 `/api` 前缀下，支持 **Session Cookie** 和 **Bearer <REDACTED>** 两种认证方式。

### 1.1 API 版本控制策略

当前 API 无显式版本前缀（如 `/api/v1`），所有端点直接在 `/api` 下。Swagger 文档版本为 `1.0.0`。

---

## 7.2 认证机制

### 2.1 Session Cookie（浏览器）

- 登录后服务端返回 `Set-Cookie: connect.sid=...`
- 浏览器自动在后续请求中携带
- 由 `validatedRequest` 中间件解析

### 2.2 Bearer <REDACTED>（API Key / JWT）

```
Authorization: Bearer <token>
```

支持的 Token 类型：
1. **API Key**：`api_keys` 表中的 `secret`
2. **JWT**：单用户模式的 AUTH_TOKEN 生成的 JWT
3. **临时 Auth Token**：`temporary_auth_tokens` 表中的 `token`

### 2.3 WebSocket 认证

WebSocket 连接通过 URL 参数 `uuid` 标识 Agent 调用，服务端在握手时验证 invocation 记录。

---

## 7.3 REST API 端点列表

### 3.1 系统端点（/api/system）

| 方法 | 路径 | 功能 | 认证 |
|------|------|------|------|
| GET | `/ping` | 健康检查 | 否 |
| GET | `/migrate` | 迁移检查 | 否 |
| GET | `/setup-complete` | 检查引导完成 | 否 |
| POST | `/login` | 用户登录 | 否 |
| POST | `/logout` | 用户登出 | 是 |
| GET | `/auth` | 检查认证状态 | 否 |
| GET | `/request-token` | 获取 JWT Token | 是 |
| POST | `/invite` | 使用邀请码注册 | 否 |
| GET | `/multi-user-mode` | 检查是否多用户模式 | 否 |
| GET | `/settings` | 获取系统设置 | 是 |
| POST | `/settings` | 更新系统设置 | 是 (admin) |
| GET | `/metrics` | 系统指标 | 是 |
| POST | `/send-feedback` | 发送反馈 | 是 |
| GET | `/logo` | 获取 Logo | 否 |
| POST | `/upload-logo` | 上传 Logo | 是 (admin) |
| DELETE | `/remove-logo` | 移除 Logo | 是 (admin) |
| GET | `/pfp` | 获取用户头像 | 是 |
| POST | `/upload-pfp` | 上传头像 | 是 |
| DELETE | `/remove-pfp` | 移除头像 | 是 |
| GET | `/api-keys` | 列出 API Keys | 是 |
| POST | `/api-keys` | 创建 API Key | 是 |
| DELETE | `/api-keys/:id` | 删除 API Key | 是 |
| GET | `/env` | 导出环境变量 | 是 (admin) |
| POST | `/env` | 更新环境变量 | 是 (admin) |
| POST | `/password-reset` | 请求密码重置 | 否 |
| POST | `/password-reset/reset` | 执行密码重置 | 否 |

### 3.2 工作空间端点（/api/workspace）

| 方法 | 路径 | 功能 | 认证 |
|------|------|------|------|
| GET | `/workspace` | 列出工作空间 | 是 |
| POST | `/workspace/new` | 创建工作空间 | 是 (admin/manager) |
| GET | `/workspace/:slug` | 获取工作空间详情 | 是 |
| POST | `/workspace/:slug` | 更新工作空间 | 是 (admin/manager) |
| DELETE | `/workspace/:slug` | 删除工作空间 | 是 (admin) |
| POST | `/workspace/:slug/upload` | 上传文档 | 是 |
| GET | `/workspace/:slug/documents` | 列出文档 | 是 |
| DELETE | `/workspace/:slug/documents/:docId` | 删除文档 | 是 |
| POST | `/workspace/:slug/stream-chat` | **流式聊天（核心）** | 是 |
| GET | `/workspace/:slug/history` | 获取聊天历史 | 是 |
| GET | `/workspace/:slug/threads` | 列出线程 | 是 |
| POST | `/workspace/:slug/threads` | 创建线程 | 是 |
| GET | `/workspace/:slug/thread/:threadSlug` | 获取线程详情 | 是 |
| POST | `/workspace/:slug/thread/:threadSlug/stream-chat` | 线程流式聊天 | 是 |
| DELETE | `/workspace/:slug/thread/:threadSlug` | 删除线程 | 是 |
| POST | `/workspace/:slug/embed` | 创建嵌入配置 | 是 (admin) |
| GET | `/workspace/:slug/embed` | 列出嵌入配置 | 是 |
| GET | `/workspace/:slug/memories` | 列出记忆 | 是 |
| POST | `/workspace/:slug/memories` | 创建记忆 | 是 |
| PUT | `/workspace/:slug/memories/:memoryId` | 更新记忆 | 是 |
| DELETE | `/workspace/:slug/memories/:memoryId` | 删除记忆 | 是 |

### 3.3 文档端点（/api/document）

| 方法 | 路径 | 功能 | 认证 |
|------|------|------|------|
| POST | `/document/upload` | 上传文档 | 是 |
| GET | `/document/:docId` | 获取文档详情 | 是 |
| DELETE | `/document/:docId` | 删除文档 | 是 |
| POST | `/document/accept-url` | 接受 URL 作为文档源 | 是 |
| POST | `/document/accept-watch-directory` | 接受热目录文档 | 是 |

### 3.4 管理员端点（/api/admin）

| 方法 | 路径 | 功能 | 认证 |
|------|------|------|------|
| GET | `/admin/users` | 列出用户 | 是 (admin) |
| POST | `/admin/users` | 创建用户 | 是 (admin) |
| PUT | `/admin/users/:id` | 更新用户 | 是 (admin) |
| DELETE | `/admin/users/:id` | 删除用户 | 是 (admin) |
| POST | `/admin/users/:id/suspend` | 暂停用户 | 是 (admin) |
| GET | `/admin/invites` | 列出邀请 | 是 (admin) |
| POST | `/admin/invites` | 创建邀请 | 是 (admin) |
| DELETE | `/admin/invites/:id` | 删除邀请 | 是 (admin) |
| GET | `/admin/statistics` | 系统统计 | 是 (admin) |

### 3.5 模型路由端点（/api/model-router）

| 方法 | 路径 | 功能 | 认证 |
|------|------|------|------|
| GET | `/model-router` | 列出路由器 | 是 (admin) |
| POST | `/model-router` | 创建路由器 | 是 (admin) |
| PUT | `/model-router/:id` | 更新路由器 | 是 (admin) |
| DELETE | `/model-router/:id` | 删除路由器 | 是 (admin) |
| POST | `/model-router/:id/rules` | 添加规则 | 是 (admin) |
| PUT | `/model-router/:id/rules/:ruleId` | 更新规则 | 是 (admin) |
| DELETE | `/model-router/:id/rules/:ruleId` | 删除规则 | 是 (admin) |

### 3.6 定时任务端点（/api/scheduled-jobs）

| 方法 | 路径 | 功能 | 认证 |
|------|------|------|------|
| GET | `/scheduled-jobs` | 列出任务 | 是 |
| POST | `/scheduled-jobs` | 创建任务 | 是 |
| PUT | `/scheduled-jobs/:id` | 更新任务 | 是 |
| DELETE | `/scheduled-jobs/:id` | 删除任务 | 是 |
| POST | `/scheduled-jobs/:id/toggle` | 启用/禁用任务 | 是 |
| GET | `/scheduled-jobs/runs/:runId` | 获取执行记录 | 是 |
| GET | `/scheduled-jobs/available-tools` | 列出可用工具 | 是 |

### 3.7 MCP Server 端点（/api/mcp-servers）

| 方法 | 路径 | 功能 | 认证 |
|------|------|------|------|
| GET | `/mcp-servers` | 列出 MCP Server | 是 (admin) |
| POST | `/mcp-servers` | 添加 MCP Server | 是 (admin) |
| PUT | `/mcp-servers/:id` | 更新 MCP Server | 是 (admin) |
| DELETE | `/mcp-servers/:id` | 删除 MCP Server | 是 (admin) |
| POST | `/mcp-servers/:id/restart` | 重启 MCP Server | 是 (admin) |

### 3.8 嵌入聊天端点（/api/embed）

| 方法 | 路径 | 功能 | 认证 |
|------|------|------|------|
| POST | `/embed/:embedId/stream-chat` | 嵌入聊天（公开） | 域名白名单 |
| GET | `/embed/:embedId/:sessionId` | 获取嵌入聊天历史 | 域名白名单 |
| DELETE | `/embed/:embedId/:sessionId` | 清除嵌入聊天历史 | 域名白名单 |

### 3.9 Agent Flow 端点（/api/agent-flows）

| 方法 | 路径 | 功能 | 认证 |
|------|------|------|------|
| GET | `/agent-flows` | 列出 Agent Flows | 是 |
| POST | `/agent-flows` | 创建 Agent Flow | 是 |
| PUT | `/agent-flows/:id` | 更新 Agent Flow | 是 |
| DELETE | `/agent-flows/:id` | 删除 Agent Flow | 是 |
| POST | `/agent-flows/:id/execute` | 执行 Agent Flow | 是 |

### 3.10 Developer API（/api/developer）

| 方法 | 路径 | 功能 | 认证 |
|------|------|------|------|
| GET | `/developer/workspaces` | 列出工作空间 | API Key |
| POST | `/developer/workspaces` | 创建工作空间 | API Key |
| GET | `/developer/workspace/:slug/chat` | OpenAI 兼容聊天 | API Key |
| POST | `/developer/workspace/:slug/chat` | OpenAI 兼容聊天 | API Key |

---

## 7.4 WebSocket API

### 4.1 Agent 实时通信

```
WSS /agent-invocation/:uuid
```

**客户端 → 服务器消息**：

| 类型 | 说明 |
|------|------|
| 工具审批 | 批准/拒绝工具调用 |
| 用户反馈 | 提供 Agent 请求的反馈 |
| 中断命令 | 终止 Agent 执行 |
| 澄清响应 | 回答 Agent 的澄清问题 |

**服务器 → 客户端消息**：

| 类型 | 说明 |
|------|------|
| `thinking` | LLM 正在思考 |
| `toolCall` | 工具调用请求 |
| `toolResult` | 工具执行结果 |
| `final` | 最终回答 |
| `error` | 执行错误 |
| `wssFailure` | WebSocket 连接失败 |

---

## 7.5 SSE 流式接口

### 5.1 聊天流式

```
POST /api/workspace/:slug/stream-chat
Content-Type: application/json
```

**请求体**：
```json
{
  "message": "什么是 RAG？",
  "attachments": []
}
```

**SSE 事件流**：
```
data: {"id": "uuid", "type": "textResponse", "textResponse": "RAG", "sources": []}

data: {"id": "uuid", "type": "textResponse", "textResponse": " 是", "sources": []}

data: {"id": "uuid", "type": "textResponse", "textResponse": "检索增强生成...", "sources": []}

data: {"id": "uuid", "type": "textResponse", "textResponse": null, "sources": [{"title": "doc.pdf", "score": 0.85}]}

data: {"id": "uuid", "type": "close"}
```

### 5.2 响应头

```
Cache-Control: no-cache
Content-Type: text/event-stream
Access-Control-Allow-Origin: *
Connection: keep-alive
```

---

## 7.6 请求/响应示例

### 6.1 登录

**请求**：
```http
POST /api/login
Content-Type: application/json

{
  "username": "admin",
  "password": "password123"
}
```

**响应**：
```http
200 OK
Set-Cookie: connect.sid=s%3A...; Path=/; HttpOnly

{
  "user": {
    "id": 1,
    "username": "admin",
    "role": "admin",
    "pfpFilename": null
  }
}
```

### 6.2 流式聊天

**请求**：
```http
POST /api/workspace/my-workspace/stream-chat
Authorization: Bearer <token>
Content-Type: application/json

{
  "message": "请总结上传的文档",
  "attachments": []
}
```

**响应**（SSE）：
```
data: {"id":"abc-123","type":"textResponse","textResponse":"根据文档内容...","sources":[]}

data: {"id":"abc-123","type":"textResponse","textResponse":"主要包含以下几个方面...","sources":[]}

data: {"id":"abc-123","type":"textResponse","textResponse":null,"sources":[{"title":"report.pdf","chunkSource":"localfile://report.pdf","score":0.92}]}

data: {"id":"abc-123","type":"close"}
```

### 6.3 上传文档

**请求**：
```http
POST /api/document/upload
Authorization: Bearer <token>
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary

------WebKitFormBoundary
Content-Disposition: form-data; name="file"; filename="report.pdf"
Content-Type: application/pdf

<binary data>
------WebKitFormBoundary--
```

**响应**：
```json
{
  "success": true,
  "document": {
    "docId": "doc-uuid",
    "filename": "report.pdf",
    "workspace": "my-workspace"
  }
}
```

---

## 7.7 错误响应格式

### 7.1 HTTP 错误

```json
{
  "error": "错误描述信息"
}
```

### 7.2 SSE 错误

```json
{
  "id": "uuid",
  "type": "abort",
  "textResponse": null,
  "sources": [],
  "close": true,
  "error": "错误描述信息"
}
```

### 7.3 状态码

| 状态码 | 含义 | 场景 |
|--------|------|------|
| 200 | 成功 | 正常响应 |
| 400 | 请求错误 | 参数缺失、消息为空 |
| 401 | 未认证 | Token 无效、Session 过期 |
| 403 | 无权限 | 角色不匹配、用户暂停 |
| 404 | 未找到 | 资源不存在 |
| 429 | 请求过多 | 超过限额 |
| 500 | 服务器错误 | 内部异常 |

---

## 7.8 Swagger/OpenAPI 文档

AnythingLLM 使用 `swagger-autogen` 自动生成 OpenAPI 3.0 文档：

- **开发环境访问**：`http://localhost:3001/api/docs`
- **生成命令**：`yarn swagger`（在 server/ 目录下）
- **输出文件**：`server/swagger/openapi.json`

**Swagger 配置**：
```javascript
const doc = {
  info: {
    version: "1.0.0",
    title: "AnythingLLM Developer API",
    description: "API endpoints for programmatic access",
  },
  host: "/api",
  schemes: ["http"],
  securityDefinitions: {
    BearerAuth: {
      type: "http",
      scheme: "bearer",
      bearerFormat: "JWT",
    },
  },
};
```

---

## 7.9 限流与安全

### 9.1 当前限流机制

| 限流点 | 实现 | 配置 |
|--------|------|------|
| 每日消息限额 | `User.canSendChat()` | 用户级 `dailyMessageLimit` |
| 嵌入聊天限额 | `canRespond()` 中间件 | 嵌入配置 `max_chats_per_day` |
| 文件上传大小 | bodyParser limit | 3GB |
| CORS | `cors({ origin: true })` | 允许所有来源 |

### 9.2 缺失的安全机制

| 机制 | 状态 | 建议 |
|------|------|------|
| 全局限流（Rate Limiting） | ❌ 无 | 添加 `express-rate-limit` |
| 请求体大小限制（除文件外） | ❌ 无 | 对非文件端点设置合理限制 |
| CSP 头 | ❌ 无 | 添加 Content-Security-Policy |
| HSTS 头 | ❌ 无 | HTTPS 时启用 |
| 审计日志 | ⚠️ 部分 | `event_logs` 表记录关键操作 |

---

**上一章 ← [06-数据模型与数据库设计](./06-数据模型与数据库设计.md)** | **下一章 → [08-部署运维与基础设施](./08-部署运维与基础设施.md)**

---

☕️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../res/promotion.jpg)