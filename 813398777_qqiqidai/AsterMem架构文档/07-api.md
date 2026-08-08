# 07 · API 设计与 AI Agent 集成

## 7.1 API 总览

AsterMem 后端暴露约 **115 个 REST API 端点**，全部定义在 `backend/web/api.py`（165KB，4487 行）。

### API 分类

| 分类 | 前缀 | 端点数 | 主要功能 |
|------|------|--------|----------|
| 认证 | `/api/auth/*` | ~8 | 登录、登出、会话、改密、重置 |
| 记忆管理 | `/api/memories*` | ~20 | CRUD、批量操作、归档、恢复 |
| 搜索 | `/api/search*` | ~6 | 关键词、语义、混合、Trunk 级 |
| 标签 | `/api/tags/*` | ~6 | 树形结构、重命名、合并 |
| 导入导出 | `/api/import*`, `/api/export*` | ~6 | ZIP、文本、URL 导入；全量导出 |
| 配置 | `/api/config*` | ~8 | Provider、搜索参数、语言 |
| 向量索引 | `/api/vector*` | ~4 | 重建、状态、Trunk 重建 |
| 时间线 | `/api/timeline/*` | ~6 | 事件 CRUD、状态更新 |
| 知识图谱 | `/api/knowledge-graph/*` | ~6 | 实体、关系、图数据 |
| 用户画像 | `/api/profile*` | ~10 | 字段读写、Claims、版本、Dream |
| 管理员 | `/api/admin/*` | ~10 | 账户、Token、日志 |
| Agent | `/api/agent/*` | ~4 | 统一工具调用入口 |
| 文件上传 | `/api/upload/*` | ~4 | 图片、音频记忆 |
| 探索 | `/api/exploration/*` | ~4 | 3D 向量空间数据 |
| Demo | `/api/demo/*` | ~4 | Demo 模式管理 |
| 其他 | `/api/*` | ~3 | 健康检查、统计、根路由 |

## 7.2 认证体系

### 双模式认证

```
┌──────────────┐        ┌──────────────────────────┐
│  Web UI 请求  │ ──────▶│ Session Cookie 认证       │
│  (浏览器)     │        │ (sessions 表, 24h 过期)   │
└──────────────┘        └──────────────────────────┘

┌──────────────┐        ┌──────────────────────────┐
│  Agent 请求   │ ──────▶│ Bearer Token 认证         │
│  (curl/SKILL)│        │ (api_tokens 表, 5级scope) │
└──────────────┘        └──────────────────────────┘

┌──────────────┐        ┌──────────────────────────┐
│  LAN 模式     │ ──────▶│ 无认证                    │
│ (可选关闭)    │        │ (login_required=false)    │
└──────────────┘        └──────────────────────────┘
```

### Token Scope 权限矩阵

| Scope | /memories (读) | /memories (写) | /config | /admin | DELETE |
|-------|:-:|:-:|:-:|:-:|:-:|
| `read` | ✅ | ❌ | ❌ | ❌ | ❌ |
| `write` | ✅ | ✅ | ❌ | ❌ | ❌ |
| `config` | ✅ | ✅ | ✅ | ❌ | ❌ |
| `admin` | ✅ | ✅ | ✅ | ✅ | ❌ |
| `destructive` | ✅ | ✅ | ✅ | ✅ | ✅ |

**默认 Token**：`read` + `write` + `config`（足以满足 Agent 日常使用）。

**Destructive 二次确认**：

```bash
# 纯删除请求 → 403 (需 destructive scope + confirm)
curl -X DELETE -H "Authorization: Bearer ast_xxx" \
  https://host/api/memories/mem_xxx

# 带 confirm → 200
curl -X DELETE -H "Authorization: Bearer ast_xxx" \
  "https://host/api/memories/mem_xxx?confirm=true"
```

## 7.3 核心端点详解

### 7.3.1 搜索 — `POST /api/search`

```json
// Request
{
    "query": "团队架构决策",
    "mode": "auto",           // auto | keyword | semantic | hybrid
    "limit": 10,
    "include_trunks": true,   // 返回段落级匹配
    "trunk_limit": 3          // 每文档返回几个 Trunk
}

// Response
{
    "query": "团队架构决策",
    "mode_used": "hybrid",    // auto 实际选用的模式
    "results": [
        {
            "memory": {
                "id": "mem_a1b2",
                "title": "团队技术决策记录",
                "tags": ["work/decisions"],
                "priority": 8,
                "match_type": "hybrid",   // keyword | semantic | hybrid
                "score": 0.0342           // RRF 分数
            },
            "trunks": [
                {
                    "id": "trunk_c3d4",
                    "order": 0,
                    "content_preview": "我们决定采用 FastAPI...",
                    "summary": "选择 FastAPI 作为后端框架",
                    "match_type": "semantic",
                    "score": 0.72         // cosine similarity
                }
            ]
        }
    ],
    "total": 3,
    "debug_info": {
        "keyword_count": 5,
        "semantic_count": 8,
        "weights": {"keyword": 1.0, "semantic": 2.0}
    }
}
```

### 7.3.2 Agent 统一入口 — `POST /api/agent/call`

这是 AI Agent 的核心调用端点：

```json
// Request
POST /api/agent/call
Authorization: Bearer ast_xxxxx
{
    "tool": "search_memories",
    "args": {
        "query": "用户的编程偏好",
        "limit": 5
    }
}
```

**支持的 tool**：

| tool | 参数 | 返回 |
|------|------|------|
| `add_memory` | title, content, tags, priority | 创建确认 + memory_id |
| `search_memories` | query, limit | 搜索结果 + Next Step Hints |
| `get_memory` | memory_id | 详情 + Trunk 列表 + 关联记忆 |
| `patch_memory` | memory_id, old_text, new_text | 精确替换确认 |
| `update_memory` | memory_id, field, value | 字段更新确认 |
| `delete_memory` | memory_id | 归档确认 |
| `list_memories` | status, limit | 记忆列表 |
| `list_memories_by_tag` | tags, limit | 按标签筛选 |
| `get_stats` | — | 统计信息 |
| `quick_search` | text | 快速语义匹配 |

### 7.3.3 配置 — `PUT /api/config`

```json
// 切换 Embedding Provider
{
    "embedding": {
        "provider_id": "openai",
        "model": "text-embedding-3-small"
    }
}
// Response: { "requires_vector_rebuild": true }

// 配置 Chat Provider
{
    "chat": {
        "provider_id": "deepseek",
        "model": "deepseek-chat"
    }
}

// 更新 API Key（写入 .env，不入 config.yaml）
{
    "providers": {
        "openai": {
            "api_key": "sk-xxxxx"     // 直接写入 .env
        }
    }
}
```

## 7.4 AI Agent 集成：SKILL 包

AsterMem 为 Cursor / Claude Code 提供了开箱即用的 **SKILL 包**（`skill/astermem/`），这是其区别于普通记忆 API 的核心价值。

### SKILL 结构

```
skill/astermem/
├── SKILL.md            # 技能定义（AI Agent 读取的指令）
├── reference.md        # 完整工具参考
└── scripts/
    ├── astermem.sh     # macOS/Linux CLI 包装
    └── astermem.ps1    # Windows PowerShell CLI 包装
```

### SKILL.md 核心设计

SKILL.md 不仅是 API 文档，更是 **AI Agent 的行为规范**：

**1. 主动记忆规则（最重要的设计）**

```markdown
### Proactive saving — the most important rule

**Always-on capture**: throughout every conversation, watch for information worth
remembering. When the user reveals any of the following, save it to AsterMem
immediately — do not wait for them to say "remember this":
- Preferences and opinions
- Personal facts (name, birthday, family, pets, location, job)
- Decisions and reasoning
- Experiences and stories
- Expertise and knowledge
- Goals and plans
- Relationships
- Recurring patterns
```

**2. 搜索规则**

```markdown
1. Search in rounds: never assume one search found everything. Search again
   with different keywords/synonyms/tags. Stop only when coverage feels sufficient.
2. Recall before write: before adding, run `quick` to check if it already exists.
3. Patch, don't overwrite: use `patch` for small corrections, not `update content`.
```

**3. 写作质量标准**

```markdown
- Titles: short and factual
- Content: Markdown format
- Tags: 2-4 hierarchical tags (e.g., people/friends, work/decisions)
- Priority: 1-10 (default 5, use 8+ only for things the user calls important)
```

### CLI 命令一览

```bash
# 搜索（推荐 quick 命令）
scripts/astermem.sh quick "<text>"                 # 语义快速匹配
scripts/astermem.sh search "<query>" [limit]       # 完整搜索

# 记忆 CRUD
scripts/astermem.sh add "<title>" "<content>" [tags] [priority]
scripts/astermem.sh get <mem_id>
scripts/astermem.sh patch <mem_id> "<old>" "<new>" # 精确替换
scripts/astermem.sh update <mem_id> <field> "<value>"
scripts/astermem.sh delete <mem_id>                 # 软删除
scripts/astermem.sh list [status] [limit]

# 标签与统计
scripts/astermem.sh tags "tag1,tag2" [limit]
scripts/astermem.sh stats

# Provider 配置
scripts/astermem.sh config [--catalog]
scripts/astermem.sh provider <id> '<json_patch>'
scripts/astermem.sh test-provider <id>

# 索引管理
scripts/astermem.sh rebuild                    # 向量重建
scripts/astermem.sh rebuild-status

# 通用 API 调用
scripts/astermem.sh api <METHOD> </api/path> ['<json>'] [confirm]
scripts/astermem.sh call <tool> '<json>'       # Agent 工具
```

### 认证流程

```bash
# 1. 首次配置
./start.sh                                    # 启动服务
# Admin → API Tokens → 创建 Token             # 获取 ast_xxxxx

# 2. 写入凭据
cat > ~/.astermem/credentials << 'EOF'
ASTERMEM_BASE_URL=http://localhost:8765
ASTERMEM_TOKEN=ast_xxxxx
EOF

# 3. 验证
scripts/astermem.sh config
```

## 7.5 API 日志与审计

所有 API 调用（包括 Agent 调用）记录到 `api_logs` 表：

| 字段 | 说明 |
|------|------|
| method | HTTP 方法 |
| path | 请求路径 |
| status_code | 响应状态码 |
| duration_ms | 耗时 |
| token_id | 调用方 Token（如果用 Token 认证） |
| ip | 请求 IP |
| timestamp | 时间戳 |

**凭据脱敏**：日志过滤器确保 API Key、Token、密码等敏感字段在记录前被 mask。

## 7.6 错误处理约定

| HTTP 状态码 | 场景 |
|-------------|------|
| 400 | 请求参数错误（缺少必填字段、格式不对） |
| 401 | 未认证（Session/Token 无效或过期） |
| 403 | 权限不足（Scope 不够） |
| 404 | 资源不存在 |
| 409 | 冲突（如用户名已存在） |
| 500 | 服务端内部错误 |

**统一错误格式**：

```json
{
    "detail": "Current password is incorrect",
    "code": "AUTH_PASSWORD_INCORRECT"
}
```

## 7.7 特殊端点

### 健康检查

```bash
GET /api/health
# → { "status": "ok", "version": "2.0.0" }
```

### 嵌入式 SPA

```
GET /                  → web-ui/dist/index.html
GET /assets/*          → Vite 构建的静态资源
GET /memories          → SPA fallback（React Router 客户端路由）
GET /api/*             → REST API（不走 SPA fallback）
```

### MCP Server

AsterMem 内置 MCP（Model Context Protocol）Server，允许 MCP 兼容的 AI 客户端直接连接，无需 HTTP API 调用。

---

*上一章：[06 · 数据模型](06-data-model.md)* · *下一章：[08 · 部署运维](08-deployment.md)*

---

☕️ 制作不易，请我喝咖啡☕️关注我➕