# 第 08 章 API 与接口设计

> **内容**: 全部 API 列表、请求/响应、认证、限流、版本控制
> **风格**: OpenAPI/Swagger 风格描述

---

## 8.1 API 架构总览

### 8.1.1 FastAPI 应用结构

```python
# lightrag/api/lightrag_server.py
app = FastAPI(
    title="LightRAG API",
    description="Simple and Fast Retrieval-Augmented Generation",
    version=__api_version__,
    docs_url=None,  # 自定义 docs 路径
    redoc_url=None,
)

# 注册中间件
app.add_middleware(CORSMiddleware, ...)

# 注册路由
app.include_router(document_router, prefix="/documents", tags=["documents"])
app.include_router(query_router, prefix="/query", tags=["query"])
app.include_router(graph_router, prefix="/graph", tags=["graph"])
app.include_router(ollama_router, prefix="/api", tags=["ollama"])
```

### 8.1.2 路由分组

| 路由前缀 | 标签 | 文件 | 端点数 |
|----------|------|------|--------|
| `/documents` | documents | `document_routes.py` | 15+ |
| `/query` | query | `query_routes.py` | 6 |
| `/graph` | graph | `graph_routes.py` | 10+ |
| `/api` | ollama | `ollama_api.py` | 5 |
| `/` | management | `lightrag_server.py` | 8+ |

### 8.1.3 API 版本

- **当前版本**: `0321`（API 版本与核心版本独立）
- **版本策略**: 语义化版本，向后兼容
- **版本标识**: 通过 `/health` 端点返回

---

## 8.2 认证与安全

### 8.2.1 认证方式

| 方式 | 请求头 | 适用场景 |
|------|--------|----------|
| **API Key** | `X-API-Key: your-key` | 服务间调用 |
| **Bearer Token** | `Authorization: Bearer <jwt>` | 用户登录后 |
| **无认证** | - | 开发环境（不推荐生产）|

### 8.2.2 JWT Token 结构

```json
{
    "sub": "user_id",
    "username": "admin",
    "roles": ["admin"],
    "exp": 1735689600,
    "iat": 1735603200
}
```

### 8.2.3 密码安全

- **哈希算法**: bcrypt
- **工作因子**: 12（默认）
- **工具**: `lightrag-hash-password` CLI

### 8.2.4 登录限流

```python
# lightrag/api/login_rate_limit.py
class LoginRateLimiter:
    """
    登录速率限制器

    限制:
    - 每分钟最大尝试次数
    - IP 级别限制
    - 账户级别限制
    """
    max_attempts_per_minute = 5
    lockout_duration = 300  # 5 分钟
```

---

## 8.3 文档管理 API

### 8.3.1 上传文档

```
POST /documents/upload
Content-Type: multipart/form-data
```

**请求参数**:

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `file` | File | 是 | 文档文件 |
| `process_options` | string | 否 | 处理选项（如 `native|F`）|

**响应示例**:
```json
{
    "track_id": "track-abc123",
    "status": "pending",
    "message": "Document uploaded successfully"
}
```

### 8.3.2 插入文本

```
POST /documents/text
Content-Type: application/json
```

**请求体**:
```json
{
    "text": "Document content here...",
    "process_options": "native|F"
}
```

### 8.3.3 文档列表

```
GET /documents/list?status=pending&page=1&page_size=20
```

**查询参数**:

| 参数 | 类型 | 说明 |
|------|------|------|
| `status` | string | 状态过滤 |
| `page` | int | 页码 |
| `page_size` | int | 每页数量 |
| `track_id` | string | 追踪 ID 过滤 |

**响应示例**:
```json
{
    "total": 100,
    "page": 1,
    "page_size": 20,
    "documents": [
        {
            "doc_id": "doc-abc123",
            "file_path": "/data/inputs/paper.pdf",
            "status": "processed",
            "content_length": 15234,
            "chunks_count": 12,
            "created_at": "2026-07-26T10:30:00Z",
            "updated_at": "2026-07-26T10:31:00Z"
        }
    ]
}
```

### 8.3.4 文档状态查询

```
GET /documents/status/{doc_id}
```

**响应示例**:
```json
{
    "doc_id": "doc-abc123",
    "status": "processed",
    "chunks_count": 12,
    "error": null,
    "track_id": "track-abc123"
}
```

### 8.3.5 文档删除

```
DELETE /documents/{doc_id}
```

**响应示例**:
```json
{
    "status": "success",
    "doc_id": "doc-abc123",
    "deleted_entities": 5,
    "deleted_relations": 3,
    "message": "Document deleted successfully"
}
```

### 8.3.6 批量上传

```
POST /documents/batch_upload
Content-Type: multipart/form-data
```

### 8.3.7 按追踪 ID 查询

```
GET /documents/track/{track_id}
```

### 8.3.8 重新处理文档

```
POST /documents/reprocess/{doc_id}
```

### 8.3.9 文档扫描

```
POST /documents/scan
```

扫描输入目录中的新文档并自动入队。

---

## 8.4 查询 API

### 8.4.1 同步查询

```
POST /query
Content-Type: application/json
```

**请求体**:
```json
{
    "query": "What is LightRAG?",
    "mode": "mix",
    "only_need_context": false,
    "only_need_prompt": false,
    "top_k": 40,
    "chunk_top_k": 40,
    "max_entity_tokens": 8192,
    "max_relation_tokens": 8192,
    "max_total_tokens": 30000,
    "enable_rerank": true,
    "response_type": "Multiple Paragraphs",
    "history_turns": 3
}
```

**响应示例**:
```json
{
    "response": "LightRAG is a simple and fast retrieval-augmented generation framework...",
    "contexts": [
        {
            "content": "LightRAG: Simple and Fast Retrieval-Augmented Generation...",
            "source": "doc-abc123",
            "chunk_id": "chunk-def456",
            "score": 0.95
        }
    ],
    "metadata": {
        "mode": "mix",
        "keywords": ["LightRAG", "RAG"],
        "entities": ["LightRAG"],
        "relations": []
    }
}
```

### 8.4.2 流式查询

```
POST /query/stream
Content-Type: application/json
```

**响应**: Server-Sent Events (SSE)

```
data: {"type": "context", "data": [...]}

data: {"type": "response", "data": "LightRAG"}

data: {"type": "response", "data": " is"}

data: {"type": "response", "data": " a..."}

data: {"type": "done"}
```

### 8.4.3 数据查询（仅检索）

```
POST /query/data
Content-Type: application/json
```

仅返回检索结果，不调用 LLM 生成回答。

### 8.4.4 查询参数完整说明

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `query` | string | - | 查询文本（必需）|
| `mode` | string | "mix" | 查询模式 |
| `only_need_context` | bool | false | 仅返回上下文 |
| `only_need_prompt` | bool | false | 仅返回提示词 |
| `top_k` | int | 40 | Top-K 检索数量 |
| `chunk_top_k` | int | 40 | 块 Top-K |
| `max_entity_tokens` | int | 8192 | 实体上下文最大 token |
| `max_relation_tokens` | int | 8192 | 关系上下文最大 token |
| `max_total_tokens` | int | 30000 | 总上下文最大 token |
| `enable_rerank` | bool | true | 启用重排序 |
| `response_type` | string | "Multiple Paragraphs" | 响应格式 |
| `stream` | bool | false | 流式输出 |
| `history_turns` | int | 3 | 历史对话轮数 |
| `top_p` | float | null | 核采样参数 |
| `temperature` | float | null | 温度参数 |

---

## 8.5 图谱 API

### 8.5.1 获取图谱标签

```
GET /graph/labels
```

**响应示例**:
```json
{
    "labels": ["Programming Language", "Framework", "Person", "Organization"]
}
```

### 8.5.2 获取实体信息

```
GET /graph/entity/{entity_name}
```

**响应示例**:
```json
{
    "entity_name": "Python",
    "entity_type": "Programming Language",
    "description": "A high-level, general-purpose programming language.",
    "source_ids": ["doc-abc123", "doc-def456"],
    "weight": 15,
    "edges": [
        {
            "target": "Guido van Rossum",
            "relationship": "created by",
            "weight": 3
        }
    ]
}
```

### 8.5.3 获取关系信息

```
GET /graph/relation/{source}/{target}
```

### 8.5.4 编辑实体

```
POST /graph/entity/edit
Content-Type: application/json
```

**请求体**:
```json
{
    "entity_name": "Python",
    "updated_data": {
        "description": "Updated description...",
        "entity_type": "Programming Language"
    }
}
```

### 8.5.5 编辑关系

```
POST /graph/relation/edit
Content-Type: application/json
```

### 8.5.6 创建实体

```
POST /graph/entity/create
```

### 8.5.7 创建关系

```
POST /graph/relation/create
```

### 8.5.8 合并实体

```
POST /graph/entity/merge
Content-Type: application/json
```

**请求体**:
```json
{
    "source_entities": ["Python", "python", "PYTHON"],
    "target_entity": "Python"
}
```

### 8.5.9 删除实体

```
DELETE /graph/entity/{entity_name}
```

### 8.5.10 删除关系

```
DELETE /graph/relation/{source}/{target}
```

### 8.5.11 获取知识图谱

```
GET /graph?max_depth=2&entity_name=Python
```

---

## 8.6 Ollama 兼容 API

### 8.6.1 Chat

```
POST /api/chat
Content-Type: application/json
```

**请求体**:
```json
{
    "model": "llama3",
    "messages": [
        {"role": "user", "content": "Hello"}
    ],
    "stream": false
}
```

### 8.6.2 Generate

```
POST /api/generate
Content-Type: application/json
```

### 8.6.3 模型列表

```
GET /api/tags
```

### 8.6.4 模型拉取

```
POST /api/pull
```

---

## 8.7 管理 API

### 8.7.1 健康检查

```
GET /health
```

**响应示例**:
```json
{
    "status": "healthy",
    "version": "1.5.5",
    "api_version": "0321",
    "storage_status": {
        "kv": "connected",
        "vector": "connected",
        "graph": "connected",
        "doc_status": "connected"
    }
}
```

### 8.7.2 系统状态

```
GET /status
```

### 8.7.3 缓存清理

```
POST /cache/clear
Content-Type: application/json
```

**请求体**:
```json
{
    "modes": ["query"]
}
```

### 8.7.4 认证状态

```
GET /auth/status
```

### 8.7.5 配置信息

```
GET /config
```

---

## 8.8 WebSocket API

### 8.8.1 实时查询

```
WS /ws/query
```

**客户端发送**:
```json
{
    "type": "query",
    "query": "What is LightRAG?",
    "mode": "mix"
}
```

**服务端推送**:
```json
{
    "type": "context",
    "data": [...]
}
```

```json
{
    "type": "response_chunk",
    "data": "LightRAG is..."
}
```

```json
{
    "type": "done"
}
```

### 8.8.2 流水线状态

```
WS /ws/pipeline
```

实时推送文档处理状态变更。

---

## 8.9 错误码与响应格式

### 8.9.1 统一错误格式

```json
{
    "error": {
        "code": "VALIDATION_ERROR",
        "message": "Invalid query parameter",
        "details": [
            {
                "field": "mode",
                "message": "must be one of: local, global, hybrid, naive, mix, bypass"
            }
        ],
        "request_id": "req-abc123"
    }
}
```

### 8.9.2 HTTP 状态码

| 状态码 | 说明 | 场景 |
|--------|------|------|
| 200 | 成功 | 正常响应 |
| 400 | 请求错误 | 参数校验失败 |
| 401 | 未认证 | Token 无效或过期 |
| 403 | 无权限 | 角色权限不足 |
| 404 | 未找到 | 资源不存在 |
| 409 | 冲突 | 资源状态冲突 |
| 422 | 无法处理 | 业务逻辑错误 |
| 429 | 请求过多 | 触发限流 |
| 500 | 服务器错误 | 内部异常 |

### 8.9.3 自定义错误码

| 错误码 | 说明 |
|--------|------|
| `STORAGE_NOT_INITIALIZED` | 存储未初始化 |
| `PIPELINE_NOT_INITIALIZED` | 流水线未初始化 |
| `DOCUMENT_NOT_FOUND` | 文档不存在 |
| `ENTITY_NOT_FOUND` | 实体不存在 |
| `INVALID_QUERY_MODE` | 无效查询模式 |
| `LLM_TIMEOUT` | LLM 调用超时 |
| `INDEX_FLUSH_FAILED` | 存储写入失败 |

---

## 8.10 OpenAPI 文档

### 8.10.1 访问方式

- **Swagger UI**: `/docs`
- **ReDoc**: `/redoc`
- **OpenAPI JSON**: `/openapi.json`

### 8.10.2 OpenAPI 信息

```yaml
openapi: 3.1.0
info:
  title: LightRAG API
  description: Simple and Fast Retrieval-Augmented Generation
  version: "0321"
  contact:
    name: LightRAG Team
    url: https://github.com/HKUDS/LightRAG
  license:
    name: MIT
    url: https://opensource.org/licenses/MIT
```

---

## 8.11 本章小结

本章对 LightRAG 的 API 设计进行了全面分析：

1. **路由结构**: 5 组路由，覆盖文档/查询/图谱/兼容/管理
2. **认证机制**: API Key + JWT + bcrypt
3. **查询 API**: 6 种模式，支持流式输出
4. **图谱 API**: 完整的 CRUD 操作
5. **Ollama 兼容**: 提供 Ollama API 兼容层
6. **错误处理**: 统一错误格式 + 自定义错误码

下一章将分析部署、运维与基础设施。

---

> **下一章**: [08-部署运维与基础设施.md](./08-部署运维与基础设施.md)

---

☕️ 制作不易，请我喝咖啡☕️关注我➕