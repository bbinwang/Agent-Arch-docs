# 05 · 核心代码走读

本章对 AsterMem 最核心的 8 个模块做逐文件、逐函数的深度走读。每个代码块都附带设计意图注释。

## 5.1 recall.py — 自适应召回（5KB，133 行）

**这是整个项目架构决策最精妙的模块。** 它替代了传统的固定阈值方案。

### 核心 API

```python
def adaptive_cutoff(
    candidates: Sequence[T],
    score_of: Callable[[T], float],
    *,
    limit: int,
    relative_ratio: float = 0.55,  # 保留分数 ≥ best × 0.55 的结果
    noise_floor: float = 0.15,     # 绝对噪声底线
    min_keep: int = 3,             # 保底返回数
) -> List[T]:
```

### 算法三步走

```python
# Step 1: 过滤噪声
above_floor = [c for c in ordered if score_of(c) >= noise_floor]

# Step 2: 相对锚点截断
cutoff = score_of(above_floor[0]) * relative_ratio  # best_hit × 0.55
kept = [c for c in above_floor if score_of(c) >= cutoff]

# Step 3: 保底机制
if len(kept) < min_keep:
    kept = above_floor[:min_keep]  # 至少返回 3 条（只要超噪声底线）
```

### 配置安全护栏

```python
MAX_NOISE_FLOOR = 0.4  # 用户配置的 noise_floor 最多 0.4

def clamp_noise_floor(value: float) -> float:
    return min(max(value, 0.0), MAX_NOISE_FLOOR)
```

**设计意图**（代码注释）：生产事故中用户配置了 0.69，但模型最高分只有 0.62，语义搜索静默返回 0 条结果。`MAX_NOISE_FLOOR=0.4` 确保即使用户误配也不会完全禁用语义搜索。

### 迁移逻辑

```python
def migrate_recall_config(config: dict) -> bool:
    """旧版 min_similarity 语义不同（绝对阈值），迁移到安全范围"""
    value = semantic.get("min_similarity")
    if value > MAX_NOISE_FLOOR:
        semantic["min_similarity"] = DEFAULT_NOISE_FLOOR  # 强制降级到 0.15
```

## 5.2 search.py — 混合搜索引擎（31KB，846 行）

### 2.1 搜索模式判定

```python
def search(self, query, mode="auto", limit=10, ...):
    if mode == "auto":
        if self.semantic_enabled and self.vector_store:
            mode = "hybrid"    # 语义可用 → 混合
        else:
            mode = "keyword"   # 否则纯关键词
```

### 2.2 双路并行检索

```python
# 关键词路径
if mode in ("keyword", "hybrid"):
    keyword_results = self._keyword_search(query, limit * 2)

# 语义路径
if mode in ("semantic", "hybrid") and self.semantic_enabled:
    # 候选池放大 3x，让 adaptive_cutoff 有足够空间找锚点
    candidates = self.vector_store.search(query, candidate_pool_size(limit), 0.0)
    matches = adaptive_cutoff(candidates, ...)
    semantic_results = ...
```

**关键细节**：`candidate_pool_size(limit) = max(limit * 3, 20)` — 向量库返回的候选数是请求量的 3 倍，给自适应截断留出锚点空间。

### 2.3 动态权重计算

```python
QUESTION_PATTERNS = [
    r'什么', r'怎么', r'如何', r'为什么', r'吗$', r'？$',
    r'\bwho\b', r'\bwhat\b', r'\bhow\b', ...
]

def _calculate_dynamic_weights(self, query: str) -> tuple:
    # 问句：措辞与原文差异大，语义主导
    if any(re.search(p, query.lower()) for p in self.QUESTION_PATTERNS):
        return (1.0, 3.0)  # keyword : semantic

    word_count = self._estimate_word_count(query)
    if word_count <= 2:    # 短词：精确名称/术语，关键词主导
        return (2.0, 1.0)
    elif word_count <= 5:  # 中等：略偏语义
        return (1.0, 2.0)
    else:                  # 长查询：描述概念，语义主导
        return (1.0, 3.0)
```

**设计约束**（注释指出）：权重差不超过 3x，避免低权重路径的排序完全失效，退化为单路径。

### 2.4 RRF 倒数排名融合

```python
def _merge_results(self, results, limit, k=60, query=""):
    W_KEYWORD, W_SEMANTIC = self._calculate_dynamic_weights(query)

    for rank, r in enumerate(keyword_results):
        rrf_scores[id] += W_KEYWORD * (1.0 / (k + rank + 1))

    for rank, r in enumerate(semantic_results):
        rrf_scores[id] += W_SEMANTIC * (1.0 / (k + rank + 1))
        # 如果两条路径都命中 → 标记为 "hybrid"
        if id in seen_objects:
            seen_objects[id].match_type = "hybrid"
```

### 2.5 Trunk 级搜索增强

Trunk（段落）级搜索额外支持 **Meta Tag 增强**：

```python
def _trunk_keyword_search(self, query, limit):
    for trunk_id, score in matches:
        # Meta Tag 匹配可额外加 30% 分数
        meta_boost = 0.0
        if trunk.meta_tags:
            meta_matches = sum(1 for k in keywords if k in meta_text)
            meta_boost = min(meta_matches * 0.1, 0.3)
        final_score = min((score / 10) + meta_boost, 1.0)
```

## 5.3 providers.py — Provider 注册表（60KB，1311 行）

这是项目最大的模块，核心是 **配置驱动 + 3 个协议适配器**。

### 3.1 Provider 目录

```python
PROVIDER_CATALOG: dict = {
    "builtin": {          # 内置 ONNX 模型（首次 ~80MB 下载）
        "api_type": "local_onnx",
        "embedding_model": "all-MiniLM-L6-v2",
    },
    "lmstudio": {         # LM Studio 本地
        "api_type": "openai_compatible",
        "base_url": "http://localhost:1234/v1",
    },
    "ollama": { ... },    # Ollama 本地
    "openai": { ... },    # OpenAI
    "anthropic": {        # Anthropic（仅 Chat）
        "api_type": "anthropic",
    },
    "google": {           # Google Gemini
        "api_type": "gemini",
    },
    "deepseek": { ... },
    "dashscope": { ... }, # 阿里通义
    "zhipu": { ... },     # 智谱 GLM
    "moonshot": { ... },  # Kimi
    # ... 共 24 种
}
```

### 3.2 三种协议适配器

```
openai_compatible:
  POST {base}/embeddings     → embedding 向量
  POST {base}/chat/completions → chat 响应
  覆盖：LM Studio, Ollama, OpenAI, DeepSeek, Moonshot, DashScope, Zhipu, ... (20种)

gemini:
  POST {base}/models/{model}:embedContent
  POST {base}/models/{model}:generateContent
  覆盖：Google AI

anthropic:
  POST {base}/messages       → chat 响应（仅 Chat，无 Embedding）
  覆盖：Anthropic Claude
```

### 3.3 API Key 安全设计

```python
def resolve_api_key(entry: dict) -> str:
    """Key 只从环境变量读取，配置文件只存 env var name"""
    env_name = (entry.get("api_key_env") or "").strip()
    if not env_name:
        return ""  # 本地 Provider（LM Studio 等）无需 Key
    return os.environ.get(env_name, "") or ""
```

**约束**：config.yaml 只存 `"api_key_env": "OPENAI_API_KEY"`，明文 Key **永不落配置文件**。PUT /config 接收的 Key 直接写入 `.env`。

### 3.4 配置迁移（normalize_config）

这个函数处理了 3 个版本的历史遗留配置迁移：

```python
def normalize_config(config: dict) -> bool:
    # v1 → v2: 清理旧版自动展开的未使用 Provider 卡片
    # v2 → v3: 统一 category 和显示名称
    # 旧 ID 映射：bailian → dashscope, googleai → google
    # active 指向已删除的 Provider → 清空选择（防止设置页无法保存）
```

**设计意图**（注释）：用户升级后配置可能不兼容新版 schema，此函数在启动时自动迁移，保证无缝升级。

### 3.5 嵌入重试策略

```python
_RETRYABLE_STATUS = frozenset({408, 409, 425, 429, 500, 502, 503, 504})

def _embedding_retry_wait(error, attempt):
    """仅重试瞬时错误，指数退避 + 抖动"""
    if isinstance(error, httpx.HTTPStatusError):
        if error.response.status_code not in _RETRYABLE_STATUS:
            return None  # 4xx（除上述）不重试
        hinted = error.response.headers.get("Retry-After")
        if hinted:
            return min(float(hinted), 30.0)
    return (attempt + 1) * 2.0 + random.uniform(0, 1)
```

## 5.4 sync.py — 多存储同步（9KB，288 行）

### 核心方法：add_memory

```python
def add_memory(self, title, content, tags=None, priority=5, source="api"):
    memory = Memory(id=generate_memory_id(), ...)

    # 1. MD 文件（同步，快）
    self.storage.save_memory(memory)

    # 2. SQLite（同步，快）
    self.database.add_memory(memory)

    # 3. 向量库（后台线程，不阻塞）
    if self.vector_store:
        _run_background(self.vector_store.add_memory, mem_copy)

    # 4. Whoosh 索引（同步）
    self.database.sync_to_whoosh(memory)

    return memory
```

### 后台线程工具

```python
def _run_background(func, *args, **kwargs):
    """守护线程执行，失败只打印警告不影响主流程"""
    def _task():
        try:
            func(*args, **kwargs)
        except Exception as e:
            print(f"[WARN] Background task failed: {e}")
    t = threading.Thread(target=_task, daemon=True)
    t.start()
```

### 文件同步（双向）

`sync_user_files()` 和 `full_sync()` 处理 MD 文件 → 数据库的同步：

```python
def sync_user_files(self):
    for memory in self.storage.scan_user_memories():
        existing = self.database.get_memory(memory.id)
        if existing:
            if memory.updated_at > existing.updated_at:
                # 文件更新 → 同步到数据库 + 向量库
                self.database.update_memory(existing)
                self.vector_store.update_memory(existing)
        else:
            # 新文件 → 添加
            self.database.add_memory(memory)
            self.vector_store.add_memory(memory)
```

## 5.5 chunker.py — AI 文档分片（16KB，487 行）

### 分片流程

```python
def chunk_document(self, document_id, content):
    # 1. 按自然段落分割
    paragraphs = self._split_into_paragraphs(content)

    # 2. AI 语义分组（可选）
    groups = None
    if self.chat_model:
        groups = self._ai_chunk(paragraphs)  # AI 决定哪些段落属于一组

    # 3. 降级：启发式分片
    if groups is None:
        groups = self._heuristic_chunk(paragraphs)  # ~800 字目标

    # 4. 合并过短组（<100 字）
    groups = self._merge_short_groups(paragraphs, groups)

    # 5. 拆分过长组（>2000 字）
    groups = self._split_long_groups(paragraphs, groups)

    # 6. 生成 Trunk 对象
    return self._create_trunks_from_groups(document_id, paragraphs, groups)
```

### AI Chunking Prompt

```python
prompt = f"""你是文档分片助手。以下是文档的编号段落列表。
请按语义分组。每组应是一个完整的主题或逻辑单元。

段落列表：
[1] 第一段内容预览...
[2] 第二段内容预览...

直接输出分组结果，格式：1-2, 3, 4-6, 7-9
- 单个段落写数字（如 3）
- 连续段落用连字符（如 1-2）
- 用逗号分隔
- 不要输出其他解释"""
```

### 幻觉容错

```python
def _parse_ai_response(self, response, total_paragraphs):
    # ... 解析 AI 输出的分组 ...
    # 处理 AI 跳过的段落：自动补为独立组
    missed = all_indices - covered
    if missed:
        # 连续的遗漏段落合并为一组
        ...
```

### Trunk 摘要与标签

```python
def generate_trunk_summary(self, trunk):
    prompt = f"""用一句话总结以下内容的核心观点（50字以内）：
    {content_preview}
    摘要："""
    response = self.chat_model.chat([...], temperature=0.3)

def generate_trunk_tags(self, trunk, existing_tags, tag_tree, similar_tags):
    tags = self.chat_model.generate_tags(
        content=trunk.content,
        existing_tags=existing_tags,    # 避免重复
        tag_tree=tag_tree,              # 约束到现有标签体系
        similar_tags=similar_tags       # 语义近邻的标签优先
    )
```

## 5.6 tools.py — Agent 工具层（45KB，1194 行）

### Next Step Hints（核心差异化设计）

```python
def _next_step_hints(self, trunk_results, seen_doc_ids):
    """生成检索导航：语义近邻 + 相关标签"""
    hints = []

    # 1) 向量邻居：找到相关但未展示的记忆
    related_docs = {}
    for r in trunk_results[:3]:
        neighbors = self.search.vector_store.find_related_trunks(
            trunk_id=r["trunk"]["id"], limit=3)
        for rel_id, _, is_same_doc in neighbors:
            if not is_same_doc and rel_id.document_id not in seen_doc_ids:
                related_docs[doc_id] = title
    if related_docs:
        hints.append(f"- 语义相似但未展示: {docs_str}, use get_memory to expand")

    # 2) 匹配 Trunk 的标签
    tags = deduplicate_tags_from_results(trunk_results)
    if tags:
        hints.append(f"- 匹配内容的标签: {tags}, use list_memories_by_tag")

    # 3) 持续提醒
    hints.append("- 单次搜索通常只覆盖一个角度。尝试不同关键词/同义词/标签继续搜索")
    return hints
```

### 无结果引导

```python
def _no_result_hints(self, query):
    lines = [
        f'No memories found related to "{query}"',
        "- 尝试更短的关键词或同义词（2-6 词核心术语推荐）",
    ]
    tags = self.sync.database.get_all_tags()
    if tags:
        # 按顶级标签分组展示（避免同层标签洪水）
        picked = one_representative_per_top_level(tags)
        lines.append(f"- 现有标签组: {picked}, use list_memories_by_tag")
    lines.append("- 或使用 list_memories 按时间浏览")
```

## 5.7 profile.py — 用户画像（28KB，647 行）

### 三层架构

```python
FIELD_SCHEMA = [
    # L1 必填
    {"key": "nickname", "required": True},
    {"key": "gender", "required": True},
    {"key": "language", "required": True},
    {"key": "timezone", "required": True},
    # L2 可选
    {"key": "occupation", "required": False},
    {"key": "location", "required": False},
    {"key": "organization", "required": False},
    {"key": "focus", "required": False},
    {"key": "preferences", "required": False},
    {"key": "taboos", "required": False},
]
```

### 字段优先级

```python
# AI 蒸馏的值（source=distilled）不会覆盖用户手动编辑的值（source=manual）
# 值变更时，旧值归档到 profile_field_history（每字段最多 10 条历史）
```

### Claim 来源锚定

```python
# profile.py 设计约束（注释）:
# "Every claim must include source memory_id; the parser rejects sourceless claims."
# "L3 AI claims: daily distillation produces claims from the **original text** of
#  changed memories (hard constraint 7.5 source anchoring)"
```

### Dream 慢循环

```python
# ProfileAuditor 两步审核：
# Step 1: claim 是否有 source 支撑？
# Step 2: claim 是否与现有 claims 矛盾？
# 通过 → active version；矛盾 → 人工裁决队列
```

## 5.8 auth.py — 认证授权（34KB，1117 行）

### 双模式认证

```python
# 1. Session Cookie（Web UI）
def verify_session(self, session_id):
    # sessions 表，24h 过期，过期自动清理

# 2. Bearer Token（API / Agent）
def verify_api_token(self, token):
    # api_tokens 表，5 级 scope
    # Token 格式：ast_{secrets.token_urlsafe(32)}
```

### 登录保护开关

```python
def is_login_required(self) -> bool:
    """默认 True。关闭后所有端点无需登录（适合可信 LAN）"""
    return bool(self.config.get("auth", {}).get("login_required", True))
```

### 密码安全

```python
def _hash_password(self, password):
    salt = self.config.get("auth", {}).get("salt", "xs_memory_salt")
    return hashlib.sha256(f"{password}{salt}".encode()).hexdigest()

def update_credentials(self, admin_id, current_password, username=None, new_password=None):
    """改密码必须验证当前密码（防越权）"""
    if not self.verify_password(admin_id, current_password):
        raise AuthError("Current password is incorrect", status=401)
```

### 离线重置

```python
def reset_to_default(self):
    """离线逃生舱：python server.py --reset-admin
    只能在本地机器执行，不暴露 HTTP 接口"""
    # 重置为 admin/admin，清除所有 session
```

---

*上一章：[04 · 模块结构](04-modules.md)* · *下一章：[06 · 数据模型](06-data-model.md)*

---

# ❤️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../res/promotion.jpg)