# 03 · 核心业务流程

本章梳理 AsterMem 的 10 大核心业务流程，配 Mermaid 时序图 / 流程图。

## 3.1 系统启动流程

```mermaid
sequenceDiagram
    participant U as 用户
    participant S as start.sh
    participant SV as server.py
    participant FA as FastAPI
    participant DB as Database
    participant CH as Chroma
    participant WH as Whoosh

    U->>S: ./start.sh
    S->>S: 检查 venv（不存在则创建）
    S->>S: 检查依赖（缺失则 pip install）
    S->>S: 检查 web-ui/dist（缺失则 npm build）
    S->>SV: exec python server.py
    SV->>SV: 加载 .env + config.yaml
    SV->>SV: 端口选择（首次随机 8000-9000）
    SV->>FA: uvicorn.run(app)

    Note over FA: startup 事件
    FA->>DB: Database(data_dir)
    DB->>DB: 建表 + 迁移（CREATE TABLE IF NOT EXISTS）
    FA->>DB: normalize_config() 迁移旧配置
    FA->>CH: VectorStore(chroma_dir)
    FA->>WH: WhooshSearch(index_dir)
    FA->>FA: 实例化 EmbeddingModel（可能 None）
    FA->>FA: 实例化 SyncManager + SearchEngine
    FA->>FA: 实例化 AuthManager（确保 admin 存在）
    FA->>FA: ProfileService.init()
    FA->>FA: ChunkingProcessor.start()
    FA->>FA: 恢复中断的分片任务
    FA-->>U: 服务就绪 :8765
```

**关键设计**：启动是幂等的——重复执行 `start.sh` 不会重复安装或构建，只启动服务。

## 3.2 记忆创建流程（多存储同步）

这是 AsterMem 最核心的数据流，展示了"一次写入、四处同步"的设计：

```mermaid
sequenceDiagram
    participant AG as Agent / API
    participant T as MemoryTools
    participant SY as SyncManager
    participant ST as MemoryStorage
    participant DB as Database
    participant VS as VectorStore
    participant WH as WhooshSearch
    participant CQ as ChunkingProcessor

    AG->>T: add_memory(title, content, tags)
    T->>SY: sync.add_memory(...)
    SY->>SY: Memory(id=generate_id(), ...)

    par 同步写入四个存储
        SY->>ST: save_memory(memory) → mem_xxxx.md
        SY->>DB: add_memory(memory) → SQLite
        SY->>WH: sync_to_whoosh(memory)
    end

    SY->>VS: _run_background(vector_store.add_memory)
    Note over VS: 后台线程，不阻塞响应

    SY-->>T: return memory
    T->>CQ: queue_document_for_chunking(memory.id)
    Note over CQ: 异步分片队列
    T-->>AG: "Added: {title} | {id}"

    Note over CQ: 后台异步执行
    CQ->>CQ: AI / 启发式分片
    CQ->>DB: 保存 Trunk 记录
    CQ->>VS: 逐 Trunk 向量化
    CQ->>CQ: AI 生成 summary + tags
    CQ->>DB: 更新 Trunk status=ready
```

**关键设计点**：

1. **响应速度优先**：MD + SQLite + Whoosh 同步写入（快），Chroma 向量化在后台线程（不阻塞）
2. **分片延迟处理**：ChunkingProcessor 异步队列处理，内容变更可触发重新分片
3. **失败隔离**：向量存储失败不影响 Memory 已落盘的数据完整性

## 3.3 混合搜索流程（核心检索路径）

```mermaid
sequenceDiagram
    participant AG as Agent
    participant T as MemoryTools
    participant SE as SearchEngine
    participant WH as WhooshSearch
    participant VS as VectorStore
    participant DB as Database
    participant RC as recall.adaptive_cutoff

    AG->>T: search_memories(query, limit=10)
    T->>SE: search_trunks(query, mode="auto")
    SE->>SE: auto → hybrid（语义启用时）

    par 关键词路径
        SE->>WH: search_trunks(query, limit*2)
        WH-->>SE: [(trunk_id, score), ...]
        SE->>DB: get_trunk(id) + get_memory(doc_id)
        SE->>SE: 计算 keyword_matches + meta_boost
    and 语义路径
        SE->>VS: search_trunks(query, limit*3, 0.0)
        VS-->>SE: candidates [(id, score)]
        SE->>RC: adaptive_cutoff(candidates)
        Note over RC: best_hit × 0.55 = cutoff<br/>保底 min_keep=3
        RC-->>SE: filtered matches
        SE->>DB: get_trunk(id) + get_memory(doc_id)
    end

    SE->>SE: _merge_trunk_results(RRF 融合)
    Note over SE: 动态权重:<br/>问句→(1,3) 短词→(2,1)<br/>Score = W × 1/(60+rank)
    SE-->>T: {results, debug_info}
    T->>T: 格式化输出 + Next Step Hints
    T-->>AG: 搜索结果 + 关联 Trunk 预览 + 下一步建议
```

**Next Step Hints 机制**（`tools.py`）：

搜索结果末尾自动附带导航建议，引导 Agent 多轮检索：

```
[Next Steps]
- Semantically similar but not shown: "团队决策记录"(mem_xxx), use get_memory to expand
- Tags from matched content: work/decisions, team, use list_memories_by_tag to broaden
- A single search usually covers only one angle. Try different keywords / synonyms / tags...
```

这是 AsterMem 区别于普通搜索 API 的关键设计——**主动引导 Agent 避免一次搜索就下结论**。

## 3.4 AI 文档分片流程（Chunking）

```mermaid
flowchart TD
    A[新 Memory 写入] --> B[ChunkingProcessor 入队]
    B --> C[取出 document_id]
    C --> D[读取完整 content]
    D --> E[_split_into_paragraphs<br/>按空行/换行分段]
    E --> F{Chat 模型可用?}
    F -->|是| G[_ai_chunk<br/>AI 语义分组]
    F -->|否| H[_heuristic_chunk<br/>按 ~800 字目标切分]
    G --> I{AI 成功?}
    I -->|否| H
    I -->|是| J[_merge_short_groups<br/>合并 <100 字的段]
    H --> J
    J --> K[_split_long_groups<br/>拆分 >2000 字的段]
    K --> L[_create_trunks<br/>生成 Trunk 对象]
    L --> M[保存 Trunk 到 SQLite<br/>status=pending]
    M --> N[逐 Trunk 向量化<br/>VectorStore.add_trunk]
    N --> O[AI 生成 summary<br/>50 字以内]
    O --> P[AI 生成 hierarchical tags]
    P --> Q[更新 Trunk status=ready]
    Q --> R{还有更多文档?}
    R -->|是| C
    R -->|否| S[等待新任务]

    style G fill:#9B59B6,color:#fff
    style H fill:#E67E22,color:#fff
    style Q fill:#50C878,color:#fff
```

**AI 分片的 Prompt 设计**（`chunker.py`）：

```
你是文档分片助手。以下是文档的编号段落列表。
请按语义分组，每组应是一个完整的主题或逻辑单元。
直接输出分组结果，格式：1-2, 3, 4-6, 7-9
```

**容错设计**：AI 输出解析时，自动处理幻觉（跳过的段落自动补为独立组）。

## 3.5 自适应召回（Adaptive Recall）

```mermaid
graph LR
    A[向量搜索候选集<br/>limit × 3 条] --> B[按 score 降序排列]
    B --> C[过滤 noise_floor 以下<br/>默认 0.15，上限 0.4]
    C --> D[cutoff = best_score × 0.55]
    D --> E[保留 score ≥ cutoff 的结果]
    E --> F{结果数 < 3?}
    F -->|是| G[强制保留 Top-3<br/>只要超过 noise_floor]
    F -->|否| H[截取 limit 条返回]
    G --> H

    style D fill:#E74C3C,color:#fff
    style G fill:#F39C12,color:#fff
```

**为什么不用固定阈值？**（`recall.py` 设计注释）

> 生产事故：用户配置 `min_similarity=0.69`，但 Embedding 模型对最精确查询的最高分仅 0.62。语义搜索静默返回 0 条结果，退化为纯关键词搜索，用户毫无感知。

自适应方案让"相关性"成为相对判断——**以当前查询的最佳匹配为锚点**，使阈值与模型和数据量无关。

## 3.6 用户画像蒸馏流程（Profile Layer + Dream）

```mermaid
sequenceDiagram
    participant DB as Database
    participant PS as ProfileService
    participant PA as ProfileAuditor
    participant LLM as Chat Provider
    participant SC as Scheduler

    Note over SC: 每日定时触发（或手动）

    SC->>PS: run_daily_distillation()
    PS->>DB: 查询今日变更的 memories
    DB-->>PS: changed_memories[]

    loop 每个 changed memory
        PS->>LLM: "从以下内容提取 claim，每条带 source memory_id"
        LLM-->>PS: claims[]
        PS->>DB: 存入候选 claims（status=pending）
    end

    Note over PA: 两步审核

    loop 每条候选 claim
        PA->>LLM: 审查 1: 是否有 source 支撑？
        LLM-->>PA: pass / fail + reason
        alt pass
            PA->>LLM: 审查 2: 与现有 claims 矛盾？
            LLM-->>PA: pass / fail
            alt pass
                PA->>DB: claim → active version
            else 矛盾
                PA->>DB: 存入矛盾队列（等待人工裁决）
            end
        else 无支撑
            PA->>DB: claim → rejected
        end
    end

    PS->>DB: 生成新 profile_version
    PS->>DB: 旧版本 → archived
```

**三层输出模型**：

| 层级 | 内容 | 数据来源 |
|------|------|----------|
| **L1 Core Traits** | nickname, gender, language, timezone（必填） | AI 蒸馏 + 用户手动编辑 |
| **L2 Optional** | occupation, location, organization, focus, preferences, taboos | AI 蒸馏 + 用户手动编辑 |
| **L3 AI Claims** | 从记忆蒸馏的事实声明，每条带 source memory_id | AI 蒸馏 + 两步审核 |

**硬约束**：每条 claim 必须包含 source memory_id，解析器拒绝无来源的 claim（`profile.py` 注释）。

## 3.7 认证与授权流程

```mermaid
flowchart TD
    Req[HTTP 请求] --> Auth{认证中间件}
    Auth -> IsPub{公开路径?<br/>/api/auth/login 等}
    IsPub -->|是| Pass[放行]
    IsPub -->|否| LoginReq{login_required?}
    LoginReq -->|否| PassNoAuth[放行<br/>用 primary_admin_id]

    LoginReq -->|是| HasSession{Session Cookie 有效?}
    HasSession -->|是| Pass
    HasSession -->|否| HasToken{Bearer Token?}
    HasToken -->|否| Reject401[401 Unauthorized]
    HasToken -->|是| VerifyToken[verify_api_token]
    VerifyToken --> Scope{需要 scope?}
    Scope -->|有权限| Pass
    Scope -->|无权限| Reject403[403 Forbidden]

    style Pass fill:#50C878,color:#fff
    style Reject401 fill:#E74C3C,color:#fff
    style Reject403 fill:#E74C3C,color:#fff
```

**Token Scope 体系**（`auth.py`）：

| Scope | 允许的操作 |
|-------|-----------|
| `read` | 搜索、统计、图谱、时间线、导出 |
| `write` | 记忆 CRUD、标签、导入、探索、时间线更新 |
| `config` | Provider 配置、语义搜索开关、索引重建 |
| `admin` | 账户管理、Token 管理、日志查看 |
| `destructive` | 删除、清空数据、重启操作 |

**默认 Token** 包含 `read` + `write` + `config`。`destructive` REST 调用需二次确认（`confirm` 参数）。

## 3.8 Agent 调用流程（/api/agent/call）

```mermaid
sequenceDiagram
    participant AG as AI Agent
    participant API as FastAPI
    participant T as MemoryTools
    participant SY as SyncManager
    participant SE as SearchEngine

    AG->>API: POST /api/agent/call<br/>Bearer ast_xxx<br/>{"tool": "search_memories", "args": {...}}

    API->>API: Token 认证 + scope 检查
    API->>T: dispatch(tool, args)

    alt tool = "add_memory"
        T->>SY: add_memory(...)
        T->>T: 触发分片回调
        T-->>API: "Added: {title} | {id}"
    else tool = "search_memories"
        T->>SE: search_trunks(query)
        T->>T: 格式化 + Next Step Hints
        T-->>API: 搜索结果文本
    else tool = "patch_memory"
        T->>SY: patch_update(exact old→new)
        T-->>API: "Updated: {title} v{version}"
    else tool = "get_memory"
        T->>SY: get_memory(id)
        T->>T: 展示 Trunk 列表 + 关联 Trunk
        T-->>API: 详情文本
    end

    API-->>AG: JSON response
```

**Agent 工具清单**（15+ 种，见 `tools.py` + SKILL）：

| 工具 | 用途 |
|------|------|
| `add_memory` | 添加记忆（触发分片） |
| `update_memory` | 更新记忆（内容变更触发重新分片） |
| `patch_memory` | 精确文本替换（优于整体更新） |
| `delete_memory` | 归档记忆（软删除） |
| `get_memory` | 获取记忆详情（含 Trunk + 关联） |
| `search_memories` | 搜索（Trunk 级 + Next Step Hints） |
| `list_memories` | 列表（按状态/来源/标签） |
| `list_memories_by_tag` | 按标签筛选 |
| `get_stats` | 统计信息 |
| `quick_search` | 快速语义匹配 |

## 3.9 文件同步流程（用户手工编辑 MD）

AsterMem 支持用户直接编辑 `./data/memories/user/` 目录下的 MD 文件：

```mermaid
sequenceDiagram
    participant U as 用户
    participant WD as watchdog Observer
    participant SY as SyncManager
    participant DB as Database
    participant VS as VectorStore

    U->>WD: 编辑/新增 .md 文件
    WD->>WD: 检测文件变更事件
    WD->>SY: sync_user_files()

    loop 每个 .md 文件
        SY->>SY: load_memory(file) → Memory 对象
        SY->>DB: get_memory(memory.id)
        alt 已存在且文件更新
            SY->>DB: update_memory(...)
            SY->>VS: update_memory(...)
        else 不存在
            SY->>DB: add_memory(...)
            SY->>VS: add_memory(...)
        end
    end

    SY-->>U: {added: N, updated: N}
```

## 3.10 向量索引重建流程

当用户切换 Embedding Provider 后，向量维度变化，必须全量重建：

```mermaid
sequenceDiagram
    participant U as 用户
    participant API as FastAPI
    participant SY as SyncManager
    participant VS as VectorStore
    participant DB as Database

    U->>API: PUT /api/config (切换 embedding_provider)
    API->>API: 检测到维度变化
    API-->>U: requires_vector_rebuild: true

    U->>API: POST /api/vector-rebuild (confirm)
    API->>SY: rebuild_all_vectors()
    SY->>VS: rebuild_index(all_memories)

    loop 每个 active memory
        VS->>VS: embedding_model.embed(text)
        VS->>VS: collection.add(...)
    end

    SY->>VS: rebuild_trunks(all_trunks)
    Note over VS: 逐 Trunk 重新向量化

    VS-->>SY: count
    SY-->>API: {vector_indexed: N}
    API-->>U: 重建完成
```

**中断恢复**：重建过程被中断（进程崩溃等），下次启动时自动检测并恢复未完成的重建。

---

*上一章：[02 · C4 架构模型](02-c4-architecture.md)* · *下一章：[04 · 模块结构](04-modules.md)*

---

☕️ 制作不易，请我喝咖啡☕️关注我➕