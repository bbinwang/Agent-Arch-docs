# 第 7 章 API 与接口设计

> 本章详细描述 WeKnora 的 238 个 HTTP API 端点（基于 Swagger/OpenAPI 规范），包括分组列表、认证方式、请求/响应示例、错误码规范。

---

## 7.1 API 总览

### 7.1.1 基本信息

| 属性 | 值 |
|------|------|
| 基础路径 | `/api/v1` |
| 认证方式 | JWT Bearer / X-API-Key / Embed Session |
| 数据格式 | JSON |
| 流式协议 | SSE (Server-Sent Events) |
| API 版本 | v1 |
| 端点总数 | 238 |
| OpenAPI 规范 | `docs/swagger.json` / `docs/swagger.yaml` |

### 7.1.2 认证方式

#### JWT Bearer（主要方式）

```
Authorization: Bearer <access_token>
```

- Access Token 有效期：15 分钟（可配置）
- Refresh Token 有效期：7 天
- 通过 `/auth/login` 获取，`/auth/refresh` 续期

#### X-API-Key（程序化访问）

```
X-API-Key: <api_key>
X-Tenant-ID: <tenant_id>  # Platform Key 时必传
```

- **Tenant Key**：固定访问所属租户
- **Platform Key**：可访问多个租户（需传 X-Tenant-ID）
- 能力级授权（`capabilities` 字段）
- 可选知识库范围限制（`scope_kb_ids`）

#### Embed Session（嵌入控件）

```
Authorization: Embed <publish_token>
X-Embed-Session: <signed_session_handle>
```

- 通过 `/embed/:channel_id/exchange` 交换会话 Token
- 域名白名单 + 三重限流（每 IP/分钟、全局/分钟、全局/天）

#### OIDC（单点登录）

```
GET /auth/oidc/url → 重定向到 OIDC Provider
GET /auth/oidc/callback?code=...&state=... → 回调获取 Token
```

#### 微信扫码

```
POST /wechat/qrcode → 获取二维码
POST /wechat/qrcode/status → 轮询扫码状态
```

### 7.1.3 统一响应格式

**成功响应**：
```json
{
    "success": true,
    "data": { ... }
}
```

**错误响应**：
```json
{
    "success": false,
    "error": {
        "code": "ERROR_CODE",
        "message": "Human readable message",
        "details": { ... }
    }
}
```

**SSE 流事件**：
```
data: {"event":"thinking","data":{...}}\n\n
data: {"event":"tool_call","data":{...}}\n\n
data: {"event":"tool_result","data":{...}}\n\n
data: {"event":"final_answer","data":{...}}\n\n
data: {"event":"done","data":{...}}\n\n
```

---

## 7.2 核心 CRUD API

### 7.2.1 认证 API（`/auth`）

| 方法 | 路径 | 认证 | 功能 | Handler |
|------|------|------|------|---------|
| POST | /auth/register | 公开 | 用户注册 | `AuthHandler.Register` |
| POST | /auth/register-by-invite | 公开（限流）| 邀请注册 | `AuthHandler.RegisterByInvite` |
| POST | /auth/invitations/lookup | 公开（限流）| 邀请查询 | `AuthHandler.LookupInvitation` |
| POST | /auth/login | 公开 | 登录获取 Token | `AuthHandler.Login` |
| POST | /auth/auto-setup | 公开 | Lite 自动初始化 | `AuthHandler.AutoSetup` |
| GET | /auth/config | 公开 | 获取认证配置 | `AuthHandler.GetAuthConfig` |
| POST | /auth/switch-tenant | Bearer | 切换租户 | `AuthHandler.SwitchTenant` |
| GET | /auth/oidc/config | 公开 | OIDC 配置 | `AuthHandler.GetOIDCConfig` |
| GET | /auth/oidc/url | Bearer | OIDC 授权 URL | `AuthHandler.GetOIDCAuthorizationURL` |
| GET | /auth/oidc/callback | 公开 | OIDC 回调 | `AuthHandler.OIDCRedirectCallback` |
| POST | /auth/refresh | 公开 | 刷新 Token | `AuthHandler.RefreshToken` |
| GET | /auth/validate | Bearer | 验证 Token | `AuthHandler.ValidateToken` |
| POST | /auth/logout | Bearer | 登出 | `AuthHandler.Logout` |
| GET | /auth/me | Bearer/API Key | 当前用户信息 | `AuthHandler.GetCurrentUser` |
| PUT | /auth/me/preferences | Bearer | 更新偏好 | `AuthHandler.UpdateMyPreferences` |
| POST | /auth/change-password | Bearer | 修改密码 | `AuthHandler.ChangePassword` |

**登录请求示例**：
```json
// POST /auth/login
{
    "email": "user@example.com",
    "password": "secure_password"
}

// Response
{
    "success": true,
    "data": {
        "access_token": "eyJhbGciOiJIUzI1NiIs...",
        "refresh_token": "eyJhbGciOiJIUzI1NiIs...",
        "expires_in": 900,
        "user": {
            "id": 1,
            "email": "user@example.com",
            "name": "User Name"
        }
    }
}
```

### 7.2.2 知识库 API（`/knowledge-bases`）

| 方法 | 路径 | 守卫 | 功能 |
|------|------|------|------|
| GET | /knowledge-bases | Viewer | 知识库列表 |
| POST | /knowledge-bases | Contributor | 创建知识库 |
| GET | /knowledge-bases/:id | Viewer + KB Access | 知识库详情 |
| PUT | /knowledge-bases/:id | OwnedKBOrAdmin + KB Write | 更新知识库 |
| DELETE | /knowledge-bases/:id | OwnedKBOrAdmin + KB Write | 删除知识库 |
| PUT | /knowledge-bases/:id/pin | Viewer + KB Read | 固定/取消固定 |
| POST | /knowledge-bases/:id/hybrid-search | Viewer + KB Read | 混合检索 |
| POST | /knowledge-bases/copy | Contributor | 克隆知识库 |
| POST | /knowledge-bases/:id/duplicate | Contributor + KB Read | 复制设置 |
| GET | /knowledge-bases/copy/progress/:task_id | Viewer | 克隆进度 |
| GET | /knowledge-bases/:id/move-targets | Viewer + KB Read | 移动目标列表 |
| GET | /knowledge-bases/:id/activity | OwnedKBOrAdmin + KB Read | 活动记录 |

**创建知识库请求示例**：
```json
// POST /knowledge-bases
{
    "name": "产品文档",
    "type": "document",
    "description": "产品相关文档知识库",
    "embedding_model_id": "model-uuid",
    "vector_store_id": "vs-uuid",
    "chunking_config": {
        "strategy": "heading",
        "chunk_size": 1000,
        "chunk_overlap": 200
    }
}
```

### 7.2.3 知识文档 API（`/knowledge` + `/knowledge-bases/:id/knowledge`）

| 方法 | 路径 | 守卫 | 功能 |
|------|------|------|------|
| POST | /knowledge-bases/:id/knowledge/file | OwnedKBOrAdmin + KB Write | 文件上传 |
| POST | /knowledge-bases/:id/knowledge/url | OwnedKBOrAdmin + KB Write | URL 导入 |
| POST | /knowledge-bases/:id/knowledge/manual | OwnedKBOrAdmin + KB Write | 手动创建 |
| GET | /knowledge-bases/:id/knowledge | Viewer + KB Read | 文档列表 |
| DELETE | /knowledge-bases/:id/knowledge | Admin + KB Write | 清空知识库 |
| GET | /knowledge/:id | Viewer + KB Read | 文档详情 |
| PUT | /knowledge/:id | OwnedKBOrAdmin + KB Write | 更新文档 |
| DELETE | /knowledge/:id | OwnedKBOrAdmin + KB Write | 删除文档 |
| POST | /knowledge/:id/reparse | OwnedKBOrAdmin + KB Write | 重新解析 |
| POST | /knowledge/:id/cancel-parse | OwnedKBOrAdmin + KB Write | 取消解析 |
| GET | /knowledge/:id/download | Contributor + KB Write | 下载原始文件 |
| GET | /knowledge/:id/preview | Viewer + KB Read | 预览文件 |
| GET | /knowledge/:id/spans | Viewer + KB Read | 解析 Trace |
| GET | /knowledge/search | Viewer | 搜索文档 |
| POST | /knowledge/batch-delete | OwnedKBOrAdmin + KB Write | 批量删除 |
| POST | /knowledge/batch-reparse | OwnedKBOrAdmin + KB Write | 批量重解析 |
| POST | /knowledge/move | OwnedKBOrAdmin + KB Write | 跨 KB 移动 |
| GET | /knowledge/batch | Viewer | 批量获取 |

### 7.2.4 FAQ API（`/knowledge-bases/:id/faq`）

| 方法 | 路径 | 守卫 | 功能 |
|------|------|------|------|
| GET | /knowledge-bases/:id/faq/entries | Viewer + KB Read | FAQ 列表 |
| POST | /knowledge-bases/:id/faq/entries | OwnedKBOrAdmin + KB Write | 批量导入 |
| POST | /knowledge-bases/:id/faq/entry | OwnedKBOrAdmin + KB Write | 创建条目 |
| GET | /knowledge-bases/:id/faq/entries/:entry_id | Viewer + KB Read | 条目详情 |
| PUT | /knowledge-bases/:id/faq/entries/:entry_id | OwnedKBOrAdmin + KB Write | 更新条目 |
| DELETE | /knowledge-bases/:id/faq/entries | OwnedKBOrAdmin + KB Write | 删除条目 |
| POST | /knowledge-bases/:id/faq/search | Viewer + KB Read | FAQ 搜索 |
| GET | /knowledge-bases/:id/faq/entries/export | Viewer + KB Read | 导出 FAQ |

### 7.2.5 标签 API（`/knowledge-bases/:id/tags`）

| 方法 | 路径 | 守卫 | 功能 |
|------|------|------|------|
| GET | /knowledge-bases/:id/tags | Viewer + KB Read | 标签列表 |
| POST | /knowledge-bases/:id/tags | OwnedKBOrAdmin + KB Write | 创建标签 |
| PUT | /knowledge-bases/:id/tags/:tag_id | OwnedKBOrAdmin + KB Write | 更新标签 |
| DELETE | /knowledge-bases/:id/tags/:tag_id | OwnedKBOrAdmin + KB Write | 删除标签 |

---

## 7.3 会话与对话 API

### 7.3.1 会话 API（`/sessions`）

| 方法 | 路径 | 守卫 | 功能 |
|------|------|------|------|
| GET | /sessions | Viewer | 会话列表 |
| POST | /sessions | Viewer | 创建会话 |
| GET | /sessions/:id | Viewer | 会话详情 |
| PUT | /sessions/:id | Viewer | 更新会话 |
| DELETE | /sessions/:id | Viewer | 删除会话 |
| DELETE | /sessions/:id/messages | Viewer | 清空消息 |
| DELETE | /sessions/batch | Viewer | 批量删除 |
| POST | /sessions/:session_id/stop | Viewer | 停止生成 |
| POST | /sessions/:session_id/pin | Viewer | 固定会话 |
| DELETE | /sessions/:id/pin | Viewer | 取消固定 |
| POST | /sessions/:session_id/title | Viewer | 生成标题 |

### 7.3.2 对话 API

| 方法 | 路径 | 守卫 | 功能 |
|------|------|------|------|
| POST | /sessions/:session_id/knowledge-qa | Viewer | RAG 问答 |
| POST | /sessions/:session_id/agent-qa | Viewer | Agent 问答 |
| GET | /sessions/continue-stream/:session_id | Viewer | SSE 流式连接 |

**问答请求示例**：
```json
// POST /sessions/{session_id}/knowledge-qa
{
    "query": "WeKnora 支持哪些向量数据库？",
    "agent_enabled": false,
    "knowledge_base_ids": ["kb-uuid-1"],
    "tag_ids": ["tag-uuid-1"],
    "model_id": "model-uuid",
    "web_search_enabled": false,
    "citation_enabled": true
}
```

**Agent 问答请求示例**：
```json
// POST /sessions/{session_id}/agent-qa
{
    "query": "总结最近上传的 3 篇文档",
    "agent_enabled": true,
    "agent_id": "agent-uuid",
    "knowledge_base_ids": ["kb-uuid-1"],
    "mentioned_items": [
        {"type": "kb", "id": "kb-uuid-1", "name": "产品文档"},
        {"type": "tag", "id": "tag-uuid-1", "kb_id": "kb-uuid-1"}
    ],
    "image_attachments": [{"data": "data:image/png;base64,..."}]
}
```

**SSE 流事件示例**：
```
data: {"event":"thinking","data":{"content":"让我思考一下..."}}

data: {"event":"tool_call","data":{"tool":"knowledge_search","args":{"queries":["向量数据库"]}}}

data: {"event":"tool_result","data":{"tool":"knowledge_search","result":"<search_results>..."}}

data: {"event":"final_answer","data":{"content":"WeKnora 支持以下向量数据库：\n1. pgvector\n2. Milvus\n..."}}

data: {"event":"done","data":{"message_id":"msg-uuid"}}
```

### 7.3.3 消息 API（`/messages`）

| 方法 | 路径 | 守卫 | 功能 |
|------|------|------|------|
| GET | /messages/:session_id/load | Viewer | 历史消息加载 |
| DELETE | /messages/:session_id/:id | Viewer | 删除消息 |
| POST | /messages/search | Viewer | 消息搜索 |
| GET | /messages/chat-history-stats | Viewer | 历史统计 |

### 7.3.4 问题建议 API

| 方法 | 路径 | 守卫 | 功能 |
|------|------|------|------|
| GET | /sessions/:id/messages/:message_id/suggestions | Viewer | 获取建议 |
| POST | /sessions/:session_id/messages/:message_id/suggestions | Viewer | 生成建议 |
| POST | /sessions/:session_id/suggestion-events | Viewer | 建议事件 |

---

## 7.4 自定义 Agent API

| 方法 | 路径 | 守卫 | 功能 |
|------|------|------|------|
| GET | /agents | Viewer (read) | Agent 列表 |
| POST | /agents | Contributor (write) | 创建 Agent |
| GET | /agents/:id | Viewer | Agent 详情 |
| PUT | /agents/:id | OwnedAgentOrAdmin | 更新 Agent |
| DELETE | /agents/:id | OwnedAgentOrAdmin | 删除 Agent |
| POST | /agents/:id/copy | Contributor | 复制 Agent |
| GET | /agents/placeholders | Viewer | 占位符定义 |
| GET | /agents/type-presets | Viewer | 类型预设 |
| GET | /agents/:id/suggested-questions | Viewer | 建议问题 |
| GET | /agents/:id/shares/:share_id | | 删除共享 |

**创建 Agent 请求示例**：
```json
// POST /agents
{
    "name": "产品问答助手",
    "description": "基于产品文档的智能问答",
    "knowledge_base_ids": ["kb-uuid-1", "kb-uuid-2"],
    "model_id": "model-uuid",
    "system_prompt": "你是产品专家...",
    "web_search_enabled": true,
    "tool_calling_enabled": true,
    "citation_enabled": true,
    "allowed_tools": ["knowledge_search", "web_search", "web_fetch"],
    "max_iterations": 10,
    "temperature": 0.7
}
```

---

## 7.5 租户管理 API

| 方法 | 路径 | 守卫 | 功能 |
|------|------|------|------|
| GET | /tenants | manage_tenant_settings | 租户列表 |
| POST | /tenants | platform manage | 创建租户 |
| GET | /tenants/:id | Viewer + PathMatch | 租户详情 |
| PUT | /tenants/:id | Owner | 更新租户 |
| DELETE | /tenants/:id | Owner | 删除租户 |
| GET | /tenants/all | CrossTenant | 全部租户 |
| GET | /tenants/search | CrossTenant | 搜索租户 |
| GET | /tenants/kv/:key | Viewer | KV 配置读取 |
| PUT | /tenants/kv/:key | Admin | KV 配置写入 |

### 7.5.1 成员管理 API

| 方法 | 路径 | 守卫 | 功能 |
|------|------|------|------|
| GET | /tenants/:id/members | Viewer (list) | 成员列表 |
| POST | /tenants/:id/members | Owner (mutate) | 添加成员 |
| PUT | /tenants/:id/members/:user_id | Owner | 更新角色 |
| DELETE | /tenants/:id/members/:user_id | Owner | 移除成员 |
| POST | /tenants/:id/leave | Viewer | 离开租户 |

### 7.5.2 邀请管理 API

| 方法 | 路径 | 守卫 | 功能 |
|------|------|------|------|
| GET | /tenants/:id/invitations | Admin | 邀请列表 |
| POST | /tenants/:id/invitations | Admin | 创建邀请 |
| DELETE | /tenants/:id/invitations/:inv_id | Admin | 撤销邀请 |
| POST | /tenants/:id/invite-links | Admin | 创建邀请链接 |
| GET | /me/invitations | Bearer | 我的邀请 |
| POST | /me/invitations/:inv_id/accept | Bearer | 接受邀请 |
| POST | /me/invitations/:inv_id/decline | Bearer | 拒绝邀请 |

### 7.5.3 API Key 管理

| 方法 | 路径 | 守卫 | 功能 |
|------|------|------|------|
| GET | /tenants/:id/api-keys | Owner | API Key 列表 |
| POST | /tenants/:id/api-keys | Owner | 创建 API Key |
| DELETE | /tenants/:id/api-keys | Owner | 删除 API Key |
| GET | /tenants/:id/api-principal-config | Owner | 主体配置 |
| PUT | /tenants/:id/api-principal-config | Owner | 更新主体配置 |
| POST | /tenants/:id/api-principal-test-token | Owner | 生成测试 Token |

**创建 API Key 请求示例**：
```json
// POST /tenants/{id}/api-keys
{
    "name": "CI Integration Key",
    "capabilities": ["retrieve", "chat", "ingest"],
    "scope_kb_ids": ["kb-uuid-1"],
    "expires_in_days": 365
}

// Response
{
    "success": true,
    "data": {
        "id": 1,
        "name": "CI Integration Key",
        "key": "wkn_xxxxxxxxxxxx",  // 仅创建时返回
        "prefix": "wkn_xxxx",
        "capabilities": ["retrieve", "chat", "ingest"],
        "created_at": "2024-01-01T00:00:00Z"
    }
}
```

---

## 7.6 向量库与存储后端 API

### 7.6.1 向量库 API（`/vector-stores`）

| 方法 | 路径 | 守卫 | 功能 |
|------|------|------|------|
| GET | /vector-stores | Viewer | 向量库列表 |
| POST | /vector-stores | Admin | 创建向量库 |
| GET | /vector-stores/:id | Viewer | 向量库详情 |
| PUT | /vector-stores/:id | Admin | 更新向量库 |
| DELETE | /vector-stores/:id | Admin | 删除向量库 |
| GET | /vector-stores/types | Viewer | 引擎类型列表 |
| POST | /vector-stores/test | Admin | 测试连接（原始）|
| POST | /vector-stores/:id/test | Admin | 测试连接（按 ID）|

**创建向量库请求示例**：
```json
// POST /vector-stores
{
    "name": "pgvector-main",
    "engine_type": "postgres",
    "connection_config": {
        "host": "localhost",
        "port": 5432,
        "database": "weknora",
        "username": "weknora",
        "password": "encrypted"
    },
    "index_config": {
        "type": "hnsw",
        "m": 16,
        "ef_construction": 200,
        "metric": "cosine"
    }
}
```

### 7.6.2 存储后端 API（`/storage-backends`）

| 方法 | 路径 | 守卫 | 功能 |
|------|------|------|------|
| GET | /storage-backends | Viewer | 后端列表 |
| POST | /storage-backends | Admin | 创建后端 |
| GET | /storage-backends/:id | Viewer | 后端详情 |
| PUT | /storage-backends/:id | Admin | 更新后端 |
| DELETE | /storage-backends/:id | Admin | 删除后端 |
| PUT | /storage-backends/:id/default | Admin | 设为默认 |
| GET | /storage-backends/types | Viewer | 类型列表 |
| POST | /storage-backends/test | Admin | 测试连接 |

---

## 7.7 MCP 与 Web 搜索 API

### 7.7.1 MCP 服务 API（`/mcp-services`）

| 方法 | 路径 | 守卫 | 功能 |
|------|------|------|------|
| GET | /mcp-services | Viewer | 服务列表 |
| POST | /mcp-services | Admin | 创建服务 |
| GET | /mcp-services/:id | Viewer | 服务详情 |
| PUT | /mcp-services/:id | Admin | 更新服务 |
| DELETE | /mcp-services/:id | Admin | 删除服务 |
| POST | /mcp-services/:id/test | Admin | 测试连接 |
| GET | /mcp-services/:id/tools | Viewer | 工具列表 |
| GET | /mcp-services/:id/resources | Viewer | 资源列表 |
| PUT | /mcp-services/:id/credentials | Admin | 更新凭证 |
| DELETE | /mcp-services/:id/credentials/:field | Admin | 清除凭证 |
| GET/PUT | /mcp-services/:id/tool-approvals/:tool_name | | 工具审批 |
| POST | /mcp-services/:id/oauth/authorize-url | | OAuth 授权 URL |
| GET | /mcp-services/:id/oauth/status | | OAuth 状态 |
| DELETE | /mcp-services/:id/oauth/token | | 撤销 OAuth Token |

### 7.7.2 Web 搜索 API

| 方法 | 路径 | 守卫 | 功能 |
|------|------|------|------|
| GET | /web-search/providers | Viewer | 搜索提供商列表 |
| GET | /web-search-providers/types | Viewer | 提供商类型 |
| POST | /web-search-providers | Admin | 创建提供商 |
| GET/PUT/DELETE | /web-search-providers/:id | Admin | 提供商 CRUD |
| POST | /web-search-providers/test | Admin | 测试（原始）|

---

## 7.8 IM 与嵌入渠道 API

### 7.8.1 IM 渠道 API（`/im-channels`）

| 方法 | 路径 | 守卫 | 功能 |
|------|------|------|------|
| GET | /im-channels | Viewer | 渠道列表 |
| PUT | /im-channels/:id | Admin | 更新渠道 |
| DELETE | /im-channels/:id | Admin | 删除渠道 |
| POST | /im-channels/:id/toggle | Admin | 启用/禁用 |
| GET/POST | /im/callback/:channel_id | 公开 | IM 回调 |

### 7.8.2 嵌入渠道 API（`/embed-channels`）

| 方法 | 路径 | 守卫 | 功能 |
|------|------|------|------|
| GET | /embed-channels | Viewer | 渠道列表 |
| POST | /embed-channels | Admin | 创建渠道 |
| GET/PUT/DELETE | /embed-channels/:id | Admin | 渠道 CRUD |
| POST | /embed-channels/:id/rotate-token | Admin | 轮换 Token |

---

## 7.9 Wiki API（`/knowledgebase/:kb_id/wiki`）

| 方法 | 路径 | 守卫 | 功能 |
|------|------|------|------|
| GET | /wiki/pages | Viewer + KB Read | 页面列表 |
| POST | /wiki/pages | OwnedWikiKBOrAdmin + KB Write | 创建页面 |
| GET | /wiki/pages/:slug | Viewer + KB Read | 页面详情 |
| PUT | /wiki/pages/:slug | OwnedWikiKBOrAdmin + KB Write | 更新页面 |
| DELETE | /wiki/pages/:slug | OwnedWikiKBOrAdmin + KB Write | 删除页面 |
| GET | /wiki/index | Viewer + KB Read | 索引 |
| GET | /wiki/graph | Viewer + KB Read | 知识图谱 |
| GET | /wiki/log | Viewer + KB Read | 日志 |
| GET | /wiki/stats | Viewer + KB Read | 统计 |
| GET | /wiki/search | Viewer + KB Read | 搜索 |
| POST | /wiki/rebuild-links | OwnedWikiKBOrAdmin | 重建互链 |
| GET | /wiki/lint | Viewer + KB Read | 质量检查 |
| POST | /wiki/auto-fix | OwnedWikiKBOrAdmin | 自动修复 |
| GET | /wiki/folders | Viewer + KB Read | 文件夹列表 |
| POST | /wiki/folders | OwnedWikiKBOrAdmin | 创建文件夹 |
| GET | /wiki/issues | Viewer + KB Read | 问题列表 |

---

## 7.10 系统管理 API

### 7.10.1 系统信息（`/system`）

| 方法 | 路径 | 守卫 | 功能 |
|------|------|------|------|
| GET | /system/info | Viewer | 系统信息 |
| GET | /system/parser-engines | Viewer | 解析引擎列表 |
| POST | /system/parser-engines/check | Admin | 检查解析引擎 |
| GET | /system/storage-engine-status | Viewer | 存储引擎状态 |
| POST | /system/storage-engine-check | Admin | 检查存储引擎 |
| POST | /system/docreader/reconnect | Admin | 重连 DocReader |

### 7.10.2 平台管理（`/system/admin`）

| 方法 | 路径 | 守卫 | 功能 |
|------|------|------|------|
| POST | /system/admin/promote | SystemAdmin | 提升系统管理员 |
| POST | /system/admin/revoke | SystemAdmin | 撤销系统管理员 |
| GET | /system/admin/list | SystemAdmin | 管理员列表 |
| POST | /system/admin/users/reset-password | SystemAdmin | 重置密码 |
| GET/POST | /system/admin/api-keys | SystemAdmin | 平台 API Key |
| GET/PUT/DELETE | /system/admin/settings/:key | SystemAdmin | 平台设置 |
| GET | /system/admin/audit-log | SystemAdmin | 系统审计 |
| GET | /system/admin/runtime/queues | SystemAdmin | 运行时队列 |
| GET | /system/admin/runtime/queues/:queue/tasks | SystemAdmin | 任务列表 |
| POST | /system/admin/runtime/queues/:queue/tasks/:id/actions/:action | SystemAdmin | 任务操作 |
| DELETE | /system/admin/runtime/queues/:queue/archived | SystemAdmin | 清理归档 |

---

## 7.11 错误码规范

### 7.11.1 错误码分类

| 代码范围 | 类别 | 说明 |
|---------|------|------|
| 400xxx | 请求参数错误 | 参数缺失/格式错误/校验失败 |
| 401xxx | 认证错误 | Token 过期/无效/API Key 无效 |
| 403xxx | 权限错误 | 角色不足/资源所有权/租户访问 |
| 404xxx | 资源不存在 | 知识库/文档/会话/用户不存在 |
| 409xxx | 冲突错误 | 重复创建/并发冲突 |
| 429xxx | 限流错误 | 请求频率超限 |
| 500xxx | 服务器错误 | 内部错误/LLM 调用失败 |
| 503xxx | 服务不可用 | 依赖服务不可用 |

### 7.11.2 常见错误码

| 代码 | 消息 | 场景 |
|------|------|------|
| INVALID_REQUEST | 请求参数无效 | 参数校验失败 |
| UNAUTHORIZED | 未认证 | 无 Token 或 Token 过期 |
| INVALID_API_KEY | API Key 无效 | Key 不存在或已撤销 |
| FORBIDDEN | 权限不足 | 角色不够或资源不属于自己 |
| RESOURCE_NOT_FOUND | 资源不存在 | ID 对应的记录不存在 |
| DUPLICATE_RESOURCE | 资源重复 | 名称/邮箱已存在 |
| RATE_LIMITED | 请求过于频繁 | 触发限流 |
| LLM_ERROR | LLM 调用失败 | Provider 返回错误 |
| PARSE_ERROR | 文档解析失败 | DocReader 解析失败 |
| VECTOR_STORE_ERROR | 向量库错误 | 向量库连接/操作失败 |

### 7.11.3 错误响应示例

```json
{
    "success": false,
    "error": {
        "code": "FORBIDDEN",
        "message": "You do not have permission to access this knowledge base",
        "details": {
            "required_role": "admin",
            "current_role": "viewer",
            "resource_type": "knowledge_base",
            "resource_id": "kb-uuid"
        }
    }
}
```

---

## 7.12 限流策略

### 7.12.1 全局限流

| 范围 | 限制 | 窗口 |
|------|------|------|
| 公开认证端点 | 30 req/IP | 60s 滑动窗口 |
| 嵌入控件（每 IP）| 可配置 | 1m/1h/24h |
| 嵌入控件（全局）| 每 IP 限制 × 20 | 1m |

### 7.12.2 IM 限流

| 范围 | 限制 | 说明 |
|------|------|------|
| 每用户 | 3 请求 | 全局队列上限 |
| 每实例 | 50 请求 | 队列容量 |
| 滑动窗口 | 10 req/user | 60s 窗口 |

### 7.12.3 API Key 限流

- 支持 `throttled` 模式的 API Key
- 最后使用时间跟踪（`last_used_at`）

---

## 7.13 API 版本控制策略

| 策略 | 说明 |
|------|------|
| URL 路径版本 | `/api/v1/...`（当前唯一版本）|
| 弃用通知 | 响应头 `Deprecation: true` + `Sunset` 头 |
| 兼容性 | 同版本内向后兼容 |
| 重大变更 | 升级主版本号（v2）并维护 v1 过渡期 |

---

> **下一章**：[第 8 章 部署、运维与基础设施](./08-deploy-ops.md) — Docker、Helm、CI/CD、监控。

---

☕️ 制作不易，请我喝咖啡☕️关注我➕