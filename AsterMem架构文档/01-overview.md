# 01 · 项目概述

## 1.1 项目定位

AsterMem 是一个 **自托管的个人记忆服务**（Self-hosted Personal Memory Service），核心理念用一句话概括：

> **Your memories, on your own machine.**

它解决一个日益突出的问题：AI Agent（Cursor、Claude Code 等）每次启动会话时对你的背景一无所知。AsterMem 为 Agent 提供一个持久化的、可检索的个人知识库，让 AI 从"失忆"变成"有记忆"。

与传统笔记软件（Notion、Obsidian）或云记忆服务（mem0、Zep）相比，AsterMem 的差异化定位：

| 维度 | AsterMem | 云记忆服务 | 传统笔记 |
|------|----------|-----------|---------|
| 数据存储 | 100% 本地 | 云端 | 本地为主 |
| AI Agent 集成 | 一等公民（REST + SKILL） | API 优先 | 无或弱 |
| 搜索方式 | 混合检索（关键词+语义） | 通常仅语义 | 仅关键词 |
| 检索粒度 | 段落级（Trunk） | 文档级 | 文档级 |
| 用户画像 | 内置 Profile Layer | 无 | 无 |
| 离线运行 | ✅ 关键词+UI+存储层完全离线 | ❌ | ✅ |
| 自定义模型 | 24 种 Provider，含 LM Studio/Ollama | 锁定 | N/A |

## 1.2 核心价值

**对最终用户：**
- **数据主权** — 所有数据在 `./data/` 目录，备份 = 拷贝一个文件夹，无云同步、无遥测
- **AI 记忆** — Agent 通过单一 `POST /api/agent/call` 端点 + SKILL 包即可读写记忆
- **混合搜索** — 关键词（Whoosh + jieba 分词）和语义搜索（Chroma 向量库）协同工作
- **段落级检索** — 不只是文档级匹配，精确到 Trunk（段落分片）
- **自带模型** — 24 种 Provider 开箱即用，包括 LM Studio 和 Ollama 做本地推理

**对 AI Agent：**
- 统一入口：`/api/agent/call` 支持添加、搜索、更新、归档记忆
- Bearer Token 认证，权限分 5 级 scope（read / write / config / admin / destructive）
- 搜索结果内置 **Next Step Hints**，引导 Agent 进行多轮检索而非一次猜测
- Profile Layer 将整个记忆库蒸馏成高密度上下文，一次 `get_profile` 调用替代多次搜索

## 1.3 技术栈

### 后端

| 层 | 技术 | 版本 | 用途 |
|----|------|------|------|
| **语言** | Python | 3.11（3.10+ 兼容） | 主语言 |
| **Web 框架** | FastAPI | ≥0.104 | REST API + SPA 托管 |
| **ASGI 服务器** | Uvicorn | ≥0.24 | 单进程服务 |
| **关系数据库** | SQLite | 内置 | 记忆元数据、历史版本、实体图谱、Profile |
| **全文搜索** | Whoosh | ≥2.7.4 | 中文分词（jieba ≥0.42.1）关键词索引 |
| **向量数据库** | ChromaDB | ≥0.4.0 | 语义搜索，cosine 相似度，HNSW 索引 |
| **Markdown 解析** | python-frontmatter | ≥1.0 | YAML Front Matter 解析 |
| **HTTP 客户端** | httpx | ≥0.25 | Provider API 调用 |
| **配置** | PyYAML | ≥6.0 | config.yaml 读写 |
| **图片处理** | Pillow | ≥10.0 | 图片记忆处理 |
| **ML 辅助** | scikit-learn + umap-learn | — | 3D embedding 散点图降维 |
| **文件监控** | watchdog | ≥3.0 | Markdown 文件变更监听 |
| **环境变量** | python-dotenv | ≥1.0 | .env 加载 Provider API Key |

### 前端

| 层 | 技术 | 用途 |
|----|------|------|
| **框架** | React 18 + TypeScript | SPA |
| **构建** | Vite | 构建到 `web-ui/dist/` |
| **国际化** | 自建 i18n | 10 种语言（en/de/es/fr/pt/ru/ja/ko/zh-CN/zh-TW） |
| **页面** | React Router | 16+ 页面组件 |

### 桌面

| 层 | 技术 | 用途 |
|----|------|------|
| **框架** | Tauri | 跨平台桌面壳 |
| **语言** | Rust | Tauri 后端（sidecar 管理 + 系统托盘） |
| **打包** | PyInstaller | Python 后端打包为单二进制 |

### 基础设施

| 层 | 技术 | 用途 |
|----|------|------|
| **容器化** | Docker + docker-compose | 一键部署 |
| **CI** | GitHub Actions | 桌面应用自动构建 |
| **部署** | systemd service | Demo 部署 |

## 1.4 架构风格与设计哲学

AsterMem 的架构可以概括为 **"单体应用 + 多存储后端 + 渐进式 AI"**：

### 设计哲学一：单进程、单端口

```
一切运行在一个进程、一个端口上：
  /api/*    → REST API + Agent Channel
  /assets/* → SPA 静态资源
  /         → SPA Fallback（React Router）
```

这是刻意的约束。`backend/main.py` 的设计注释写道：

> *"Maintain the 'one command to start' deployment experience — FastAPI serves both /api/* and web-ui/dist build artifacts; frontend and backend are separate in code, but run in a single process on a single port."*

### 设计哲学二：三存储同步

每条 Memory 的写入同时触发三个存储后端：

```
Memory 写入
  ├── 1. Markdown 文件（./data/memories/api/mem_xxxx.md） — 人类可读，备份友好
  ├── 2. SQLite（./data/memories.db） — 结构化查询、元数据、关系
  └── 3. Chroma 向量库（./data/chroma/） — 语义搜索
       └── 4. Whoosh 索引（./data/whoosh_index/） — 关键词全文搜索
```

四个存储各司其职，通过 `SyncManager` 统一同步。

### 设计哲学三：优雅降级

语义搜索依赖外部 Provider（Embedding 模型），但系统设计确保：

- **Provider 不可用** → 语义搜索自动降级为关键词搜索，绝不阻止启动
- **向量索引重建中断** → 下次启动自动恢复（resume_incomplete_rebuild）
- **Chroma 集合丢失** → 自动重建（_ensure_collection）
- **AI Chunking 失败** → 降级为启发式分片（_heuristic_chunk）

### 设计哲学四：配置驱动 Provider

24 种模型提供商的差异全部封装在配置中，代码层只有 3 个协议适配器：

```
openai_compatible  → LM Studio / OpenAI / OpenRouter / DeepSeek / Moonshot / ... (20种)
gemini             → Google AI 原生 (:embedContent / :generateContent)
anthropic          → Anthropic Messages API
```

API Key 只存在 `.env` 和环境变量中，config.yaml 只记录环境变量名，**明文密钥永不落配置文件**。

### 设计哲学五：自适应召回（Adaptive Recall）

这是一个关键的架构决策，记录在 `memory/recall.py` 中：

传统方案用一个全局绝对阈值（如 `min_similarity=0.69`）判断"什么算相关"。但这个值：
- 不可移植——换一个 Embedding 模型，整个 cosine 分布就变了
- 用户无法知道该填多少
- 实际线上事故：0.69 的阈值让语义搜索永远返回 0 条结果，静默退化为纯关键词搜索

AsterMem 的方案：**以当前查询的最佳匹配为锚点，保留分数足够接近的结果**。绝对分数只用于过滤纯噪声（noise_floor ≤ 0.4）。这让阈值与模型和数据量无关。

```python
# recall.py 核心逻辑
cutoff = best_hit_score * relative_ratio  # 默认 0.55
kept = [c for c in candidates if c.score >= cutoff]
# 保底：至少返回 min_keep 条（默认 3），防止查询措辞偏离时结果太少
```

## 1.5 关键功能特性

### 检索相关

- **混合搜索**：关键词（Whoosh + jieba）+ 语义（Chroma）并行检索，RRF 倒数排名融合
- **动态权重**：根据查询特征自动调整关键词/语义权重（问句偏语义，短词偏关键词）
- **段落级检索（Trunk）**：文档被 AI 分片为语义段落，检索精确到段落
- **Meta Tag 增强**：AI 从 Trunk 中抽取隐式标签，提升搜索召回率
- **Next Step Hints**：搜索结果附带"下一步"建议，引导 Agent 多轮检索

### AI Agent 相关

- **统一 Agent 端点**：`POST /api/agent/call`，支持 15+ 种工具（add/search/patch/list...）
- **SKILL 包**：Cursor / Claude Code 一键集成，`scripts/astermem.sh` 封装所有操作
- **Token 权限体系**：5 级 scope，destructive 操作需二次确认
- **主动记忆**：SKILL 的设计理念是 Agent 应主动记忆用户信息，不等用户说"记住这个"

### 数据管理

- **多模态记忆**：文本、图片（含 OCR + EXIF + AI 描述）、音频
- **版本历史**：每次更新保存历史版本（memory_history 表）
- **知识图谱**：实体抽取 + 关系三元组（entity / entity_relations 表）
- **时间线**：从记忆中抽取时间事件（time_events 表），支持 todo/calendar 视图
- **导入导出**：ZIP 格式，包含所有 MD 文件 + index.json
- **用户画像层（Profile Layer）**：
  - L1/L2 结构化字段（nickname / gender / language / timezone / occupation...）
  - L3 AI Claims（从记忆蒸馏而来，带来源追溯）
  - Dream 慢循环（夜间运行 LLM 整理，产出候选 claim）

### 用户体验

- **10 种语言** UI
- **单命令启动**：`./start.sh`（幂等：首次创建 venv、安装依赖、构建 UI、启动服务）
- **随机端口**：首次启动从 8000-9000 随机选端口（避开 5000），写入 config.yaml 后固定
- **一键 Docker**：`docker compose up -d`，固定端口 8768

## 1.6 非功能性需求

| 维度 | 实现 |
|------|------|
| **安全性** | SHA-256 密码哈希（salted）、Bearer Token 认证、凭据脱敏日志、目录遍历防护 |
| **可用性** | 优雅降级（语义→关键词）、自动恢复中断任务、单命令启动 |
| **可维护性** | 配置驱动 Provider、模块化设计、Background-Design-Constraint 注释风格 |
| **可扩展性** | Provider 注册表（新增 Provider 零代码改动）、Trunk Meta 扩展 |
| **性能** | 后台向量索引、RRF 高效合并、Whoosh 本地索引、候选池 3x 放大 |
| **隐私性** | 零遥测、零云同步、所有数据本地、明文密钥不入配置文件 |

## 1.7 项目规模

| 维度 | 数量 |
|------|------|
| Python 后端模块 | ~35 个（`backend/memory/` + `backend/web/`） |
| TypeScript 前端组件 | ~40 个（`web-ui/src/`） |
| REST API 端点 | ~115 个 |
| 测试文件 | 16 个（pytest） |
| 内置 Provider | 24 种 |
| 支持 UI 语言 | 10 种 |
| 代码行数（估） | ~15,000 行（Python ~10k + TS ~5k） |
| 依赖包 | 20 个（Python） |

---

*下一章：[02 · C4 架构模型](02-c4-architecture.md)*

---

☕️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../res/promotion.jpg)