# 1. 项目概述 (Project Overview)

> **章节编号**: 01/13  
> **预计篇幅**: ~8,000 字  
> **关联文件**: `README.md`, `requirements.txt`, `docker-compose.yml`, `dockerfile`

---

## 1.1 项目背景与目标

### 1.1.1 项目起源

本项目名为 **"Sophisticated Controllable Agent for Complex RAG Tasks"**，由 **Nir Diamant**（DiamantAI）创建并维护，是其在《RAG Made Simple》一书中核心思想的代码实践。项目仓库地址为：`https://github.com/NirDiamant/Controllable-RAG-Agent`。

项目的核心洞察是：**传统 RAG 系统依赖语义相似度检索，只能回答"检索即答案"的浅层问题，无法处理需要多跳推理、信息聚合、因果推断的复杂问题**。例如：

- ❌ 传统 RAG 难以回答：*"帮助反派的那位教授教的是什么课？"*（需要识别反派 → 识别帮助者 → 识别课程）
- ✅ 本项目可以回答：通过 **计划-执行-重规划（Plan-and-Execute）** 循环，将问题分解为多步子任务，逐步检索、蒸馏、验证、聚合，最终推理出答案。

### 1.1.2 核心价值主张

| 价值维度 | 传统 RAG | 本项目的可控 Agent RAG |
|----------|----------|------------------------|
| **问题复杂度** | 单跳、直接匹配 | 多跳、推理、聚合 |
| **检索策略** | 单一向量检索 | 三路异构检索（chunks/summaries/quotes） |
| **幻觉控制** | 无或简单重排序 | 多层验证（相关性、基于事实、蒸馏一致性） |
| **执行透明度** | 黑盒 | 白盒可视化（Streamlit 实时图状态） |
| **计划能力** | 无 | LLM 驱动的动态计划与重规划 |
| **偏差控制** | 无 | 命名实体匿名化消除 LLM 先验知识 |

### 1.1.3 解决的问题域

本项目解决的是 **Complex RAG** 场景下的核心挑战：

1. **多跳推理（Multi-hop Reasoning）**: 问题需要跨越多个文档片段进行推理
2. **信息聚合（Information Aggregation）**: 答案散布在多个章节，需要汇总
3. **因果链推断（Causal Inference）**: 需要理解事件之间的因果关系
4. **幻觉防控（Hallucination Prevention）**: 确保答案严格基于检索到的数据
5. **可解释性（Explainability）**: 每一步决策都可追溯、可视化

### 1.1.4 目标用户

| 用户类型 | 使用场景 | 核心诉求 |
|----------|----------|----------|
| **AI 研究员** | 学习 Agentic RAG 前沿架构 | 理解 Plan-and-Execute + 图编排 |
| **LLM 应用工程师** | 构建企业级 RAG 系统 | 可复用、可扩展的 Agent 模板 |
| **技术教育者** | 教学 RAG 高级技术 | 清晰的代码 + 可视化演示 |
| **数据科学家** | 评估 RAG 质量 | Ragas 指标集成与评估流程 |
| **技术管理者** | 评估 RAG 方案可行性 | 了解能力边界与局限性 |

---

## 1.2 技术栈完整清单

### 1.2.1 编程语言与运行时

| 类别 | 技术 | 版本 | 用途 |
|------|------|------|------|
| **主语言** | Python | 3.11（Docker 镜像）/ 3.8+（本地） | 全栈开发 |
| **Notebook** | Jupyter | ipykernel 6.29.4 | 教程与实验 |

### 1.2.2 AI / LLM 框架生态

| 类别 | 技术 | 版本 | 核心作用 |
|------|------|------|----------|
| **Agent 编排** | LangGraph | 0.0.49 | 状态图编排，定义 Agent 的"大脑" |
| **LLM 抽象层** | LangChain | latest | Prompt 模板、Chain 抽象、Document 模型 |
| **社区扩展** | langchain-community | latest | 向量存储、Loader 集成 |
| **核心库** | langchain-core | latest | BaseModel、OutputParser、Runnable |
| **OpenAI 集成** | langchain-openai | latest | ChatOpenAI、OpenAIEmbeddings |
| **Groq 集成** | langchain-groq | latest | ChatGroq（备选 LLM 后端） |
| **文本分割** | langchain-text-splitters | latest | RecursiveCharacterTextSplitter |
| **Hub** | langchainhub | latest | Prompt 共享（可选） |
| **可观测性** | langsmith | 0.1.125 | 链路追踪、调试 |

### 1.2.3 向量存储与检索

| 类别 | 技术 | 版本 | 核心作用 |
|------|------|------|----------|
| **向量数据库** | FAISS (cpu) | 1.8.0 | 本地向量相似度检索 |
| **嵌入模型** | OpenAI Embeddings | via API | 文本向量化（text-embedding-3） |
| **备选索引** | hnswlib | 0.8.0 | HNSW 近似最近邻（FAISS 内部可用） |

### 1.2.4 数据处理与文档处理

| 类别 | 技术 | 版本 | 核心作用 |
|------|------|------|----------|
| **PDF 读取** | pypdf | 4.2.0 / PyPDF2 3.0.1 | PDF 文本提取 |
| **Token 计数** | tiktoken | 0.7.0 | 精确计算 GPT token 数 |
| **NLP 工具** | pysbd | 0.3.4 | 句子边界检测 |
| **LCS 算法** | pylcs | 0.1.1 | 最长公共子序列（相似度计算） |
| **数据处理** | pandas | 2.2.2 | 评估结果分析 |
| **数据序列化** | dill | latest | Python 对象序列化 |
| **数据校验** | pydantic | >=1.9.0 | 结构化输出 schema |

### 1.2.5 前端与可视化

| 类别 | 技术 | 版本 | 核心作用 |
|------|------|------|----------|
| **Web UI** | Streamlit | 1.32.0 | 实时 Agent 执行可视化 |
| **图可视化** | pyvis | 0.3.2 | 交互式网络图（节点高亮） |
| **组件嵌入** | streamlit.components.v1 | - | HTML/iframe 嵌入 |
| **备选可视化** | graphviz | 0.20.3 | 静态图渲染 |
| **备选可视化** | networkx | 3.3 | 图算法 |

### 1.2.6 评估与质量

| 类别 | 技术 | 版本 | 核心作用 |
|------|------|------|----------|
| **RAG 评估** | ragas | 0.1.7 | 多维度 RAG 质量指标 |
| **评估维度** | faithfulness, answer_relevancy, context_recall, answer_similarity, answer_correctness | - | 5 项核心指标 |

### 1.2.7 可观测性与遥测

| 类别 | 技术 | 版本 | 核心作用 |
|------|------|------|----------|
| **Traceloop** | traceloop-sdk | 0.23.0 | LLM 应用可观测性 |
| **OpenTelemetry** | opentelemetry-* | 1.25.0 | 分布式追踪标准 |
| **PostHog** | posthog | 3.5.0 | 产品分析（可选） |

### 1.2.8 部署与基础设施

| 类别 | 技术 | 版本 | 核心作用 |
|------|------|------|----------|
| **容器化** | Docker | - | 应用打包 |
| **编排** | Docker Compose | 3.8 | 单容器编排 |
| **Web 服务器** | Streamlit | 内置 | HTTP 服务 |
| **包管理** | pip | - | Python 依赖 |
| **环境管理** | python-dotenv | 1.0.1 | .env 文件加载 |

### 1.2.9 完整依赖分类统计

```
requirements.txt 依赖分类:
├── AI/LLM 核心: langgraph, langchain*, openai, groq, anthropic
├── 向量/检索: faiss-cpu, hnswlib, docarray
├── 文档处理: pypdf, PyPDF2, tiktoken, pysbd, pylcs
├── 可视化: streamlit, pyvis, graphviz, networkx, bokeh, panel
├── 评估: ragas, datasets
├── 数据处理: pandas, numpy, pyarrow
├── 可观测性: opentelemetry-*, traceloop-sdk, posthog
├── 工具: requests, httpx, tqdm, pyyaml, python-dotenv, dill
└── Jupyter: ipykernel, ipython, jupyter_*, ipywidgets
```

---

## 1.3 项目架构风格与理由

### 1.3.1 总体架构风格：分层状态图驱动的智能体架构

本项目采用 **"分层状态图驱动的智能体架构"（Layered State-Graph-Driven Agent Architecture）**，融合了多种架构模式：

```
┌─────────────────────────────────────────────────────────────┐
│                   Layer 4: 可视化层 (Streamlit)               │
│                   Web UI + 实时图状态可视化                    │
├─────────────────────────────────────────────────────────────┤
│                   Layer 3: 编排层 (LangGraph)                 │
│                   主图 (Plan-and-Execute)                     │
│                   ┌─────────┬──────────┬──────────┐          │
│                   │ 子图1   │ 子图2    │ 子图3    │          │
│                   │ Chunks  │ Summaries│ Quotes   │          │
│                   └─────────┴──────────┴──────────┘          │
├─────────────────────────────────────────────────────────────┤
│                   Layer 2: 能力层 (Chains)                    │
│                   12 条独立 LLM Chain                         │
│                   (规划/检索/蒸馏/验证/回答)                   │
├─────────────────────────────────────────────────────────────┤
│                   Layer 1: 基础设施层                         │
│                   FAISS / OpenAI API / PDF Loader             │
└─────────────────────────────────────────────────────────────┘
```

### 1.3.2 架构风格选择理由

| 架构决策 | 选择 | 理由 |
|----------|------|------|
| **Agent 模式** | Plan-and-Execute | 复杂问题需要显式分解，单步 ReAct 无法保证全局最优 |
| **编排方式** | LangGraph 状态图 | 确定性执行、可视化友好、支持循环与条件分支 |
| **子图设计** | 三段式检索子图 | 检索-蒸馏-验证闭环，确保检索质量 |
| **LLM 调用** | Chain 抽象（LCEL） | 声明式组合、易于替换模型、支持流式输出 |
| **状态管理** | TypedDict 状态对象 | 类型安全、图节点间状态传递清晰 |
| **检索策略** | 三路异构向量存储 | 不同粒度信息（细节/摘要/引文）互补 |

### 1.3.3 设计模式应用

| 设计模式 | 应用位置 | 说明 |
|----------|----------|------|
| **State 模式** | `PlanExecute` TypedDict | Agent 状态机，每个节点修改状态 |
| **Strategy 模式** | 4 种检索工具 (A/B/C/D) | TaskHandler 动态选择策略 |
| **Chain of Responsibility** | LangGraph 边路由 | 条件边将状态传递给不同处理器 |
| **Template Method** | 子图结构统一 | 三段式检索子图共享相同结构 |
| **Factory 模式** | `create_*_chain()` 函数 | 工厂函数创建可复用的 LLM Chain |
| **Pipeline 模式** | Notebook 数据处理 | 加载→分块→摘要→编码的流水线 |
| **Observer 模式** | Streamlit 状态监听 | 图状态变化触发 UI 更新 |

### 1.3.4 架构约束与原则

1. **确定性优先**: 核心图逻辑是确定性的，LLM 仅用于决策节点
2. **单数据源**: 所有知识来自向量存储，不依赖 LLM 参数知识
3. **无状态服务**: 每次请求独立执行，不维护会话状态
4. **同步执行**: 图节点同步执行（simulate_agent.py），Notebook 中可流式
5. **最小权限**: 仅需 OpenAI/Groq API Key，无额外基础设施

---

## 1.4 关键功能特性

### 1.4.1 核心功能矩阵

| 功能编号 | 功能名称 | 功能描述 | 关键实现 |
|----------|----------|----------|----------|
| F-01 | **命名实体匿名化** | 将问题中的人名/地名替换为变量 X/Y/Z | `anonymize_question_chain` |
| F-02 | **高-level 计划生成** | 为匿名化问题生成步骤化计划 | `planner` (Plan chain) |
| F-03 | **计划去匿名化** | 将变量替换回原始实体 | `de_anonymize_plan_chain` |
| F-04 | **计划细化** | 将计划步骤映射到可执行任务类型 | `break_down_plan_chain` |
| F-05 | **任务路由** | 根据任务类型选择检索工具或回答 | `task_handler_chain` |
| F-06 | **Chunks 检索** | 从书块向量存储检索相关段落 | 子图 1 |
| F-07 | **Summaries 检索** | 从章节摘要向量存储检索 | 子图 2 |
| F-08 | **Quotes 检索** | 从引文向量存储检索相关引文 | 子图 3 |
| F-09 | **内容蒸馏** | 过滤检索结果，仅保留相关部分 | `keep_only_relevant_content_chain` |
| F-10 | **蒸馏验证** | 验证蒸馏内容是否基于原始检索 | `is_distilled_content_grounded_on_content_chain` |
| F-11 | **CoT 回答生成** | 基于上下文使用链式推理回答 | `question_answer_from_context_cot_chain` |
| F-12 | **幻觉检测** | 验证答案是否基于上下文（防幻觉） | `is_grounded_on_facts_chain` |
| F-13 | **可回答性判断** | 判断当前聚合上下文是否足够回答 | `can_be_answered_already_chain` |
| F-14 | **动态重规划** | 根据已完成步骤和聚合信息更新计划 | `replanner` |
| F-15 | **最终答案生成** | 使用全部聚合上下文生成最终回答 | `run_qualtative_answer_workflow_for_final_answer` |
| F-16 | **实时可视化** | Streamlit 界面实时显示图执行状态 | `simulate_agent.py` |
| F-17 | **质量评估** | 使用 Ragas 指标评估 RAG 质量 | Notebook 评估单元 |

### 1.4.2 功能特性详解

#### F-01 ~ F-05: 计划流水线

```
原始问题 → 匿名化 → 计划生成 → 去匿名化 → 细化 → [任务1, 任务2, ...]
```

这一流水线确保：
- **消除 LLM 先验偏见**: 匿名化后 LLM 不知道 "Harry Potter" 是谁，只能基于问题结构制定计划
- **结构化执行**: 每个任务明确是可检索（A/B/C）还是可回答（D）

#### F-06 ~ F-10: 检索与蒸馏

```
查询 → 向量检索 → 原始上下文 → LLM 蒸馏 → 相关上下文 → 验证 → 聚合
```

每个检索子图都是一个 **质量闭环**：
1. 检索器返回 top-k 最相似文档
2. LLM 蒸馏器过滤掉无关句子
3. 验证器确保蒸馏内容没有"编造"
4. 未通过验证则重新蒸馏（循环）

#### F-11 ~ F-12: 回答与幻觉防控

```
问题 + 上下文 → CoT 生成答案 → 幻觉检测 → (通过) 输出 / (失败) 重新生成
```

这是项目的**核心创新点之一**：不仅生成答案，还验证答案是否"基于事实"。

#### F-13 ~ F-15: 循环与终止

```
[任务队列] → 执行 → 聚合上下文 → 可回答? → (是) 最终回答 / (否) 继续执行
```

通过 `can_be_answered_already` 判断是否需要更多检索，实现**自适应检索深度**。

---

## 1.5 非功能性需求

### 1.5.1 性能 (Performance)

| 指标 | 目标值 | 实际表现 | 瓶颈分析 |
|------|--------|----------|----------|
| **单次问题延迟** | < 60s | 30-120s（取决于计划步数） | LLM 调用次数（每步 1-3 次） |
| **检索延迟** | < 500ms | ~100ms（FAISS 本地） | 向量维度（1536 维） |
| **嵌入调用** | - | 每次检索 1 次 API 调用 | OpenAI API 速率限制 |
| **并发用户** | 1-5 | 单 Streamlit 实例 | 无异步架构 |
| **内存占用** | < 2GB | ~1.5GB（3 个 FAISS 索引） | 向量存储大小 |

**性能特征**:
- **I/O 密集型**: 主要延迟来自 LLM API 调用（每问题 10-30 次调用）
- **无 GPU 依赖**: FAISS CPU 版本即可满足小规模数据
- **可缓存**: 向量存储本地持久化，避免重复编码

### 1.5.2 扩展性 (Scalability)

| 维度 | 当前状态 | 扩展路径 |
|------|----------|----------|
| **数据规模** | 单本书（~7 万字） | 可扩展至多本书，需分库或重分块 |
| **用户规模** | 单机单用户 | 需改造为异步 + 多 worker |
| **模型扩展** | OpenAI/Groq | 可接入任何 LangChain 支持的 LLM |
| **检索维度** | 3 路（chunks/summaries/quotes） | 可增加新类型（如 characters, events） |
| **图复杂度** | 11 节点主图 | LangGraph 支持任意复杂度 |

**扩展性瓶颈**:
- **LLM 成本**: 每问题约 10-30 次 API 调用，成本较高
- **上下文窗口**: 聚合上下文可能超出 LLM 上下文限制
- **串行执行**: 当前为串行，未利用并行检索

### 1.5.3 安全性 (Security)

| 安全维度 | 当前措施 | 风险等级 | 改进建议 |
|----------|----------|----------|----------|
| **API Key 管理** | .env 文件 + 环境变量 | 中 | 使用 Secret Manager |
| **Prompt 注入** | 无显式防护 | 高 | 增加输入校验与 Prompt 隔离 |
| **数据安全** | 本地向量存储 | 低 | 加密存储 |
| **网络安全** | 本地运行 | 低 | 部署时增加 TLS |
| **反序列化** | `allow_dangerous_deserialization=True` | 高 | 使用受信任数据源 |

**关键安全注意**:
- `allow_dangerous_deserialization=True` 是一个安全风险，生产环境应确保向量存储来自受信任源
- Prompt 注入风险：用户问题直接传入 LLM，可能被恶意构造

### 1.5.4 可用性 (Availability)

| 指标 | 目标 | 当前 | 说明 |
|------|------|------|------|
| **SLA** | 99.9% | N/A | 本地应用，无 SLA 承诺 |
| **故障恢复** | 自动重试 | 无 | 需增加重试机制 |
| **降级策略** | 有 | 无 | 无 LLM 降级方案 |
| **健康检查** | 有 | 无 | Docker 可增加 healthcheck |

### 1.5.5 可维护性 (Maintainability)

| 维度 | 评分 | 说明 |
|------|------|------|
| **代码结构** | ⭐⭐⭐⭐ | 模块化清晰，函数职责单一 |
| **文档** | ⭐⭐⭐ | README 详尽，但缺少 docstring |
| **测试** | ⭐⭐ | 仅有评估，无单元测试 |
| **日志** | ⭐⭐ | print 语句，无结构化日志 |
| **配置管理** | ⭐⭐⭐ | .env + 环境变量 |
| **类型注解** | ⭐⭐⭐⭐ | TypedDict + Pydantic 广泛使用 |

### 1.5.6 可观测性 (Observability)

| 维度 | 当前状态 | 工具 |
|------|----------|------|
| **日志** | print 输出 | 标准输出 |
| **追踪** | LangGraph 流式输出 | LangSmith（可选） |
| **指标** | Ragas 评估 | ragas 库 |
| **可视化** | Streamlit 实时图 | pyvis + Streamlit |
| **告警** | 无 | - |

### 1.5.7 可移植性 (Portability)

| 维度 | 状态 | 说明 |
|------|------|------|
| **容器化** | ✅ | Dockerfile + docker-compose.yml |
| **跨平台** | ✅ | Python 跨平台，Windows 需 pywin32 |
| **云就绪** | 部分 | 可部署到任何 Docker 环境 |
| **环境隔离** | ✅ | Docker 镜像隔离 |

---

## 1.6 项目演示数据集

### 1.6.1 数据集选择理由

项目选择 **《哈利·波特与魔法石》（Harry Potter Book 1）** 作为演示数据集，原因包括：

1. **广为人知**: 便于验证模型是否依赖参数知识 vs 检索知识
2. **复杂叙事**: 多角色、多事件、因果链，适合测试多跳推理
3. **版权灰色**: 个人学习使用，非商业分发
4. **规模适中**: ~7 万字，适合快速实验

### 1.6.2 数据规模

| 数据类型 | 数量 | 存储大小 | 说明 |
|----------|------|----------|------|
| **章节数** | 17 章 | - | 按 "CHAPTER X" 正则分割 |
| **Chunks** | ~700 个 | 3.8 MB (faiss) | 1000 字符/块，200 重叠 |
| **章节摘要** | 17 个 | 104 KB (faiss) | 每章一个 LLM 生成摘要 |
| **引文** | ~500 条 | 8 MB (faiss) | 正则提取 ≥50 字符的引号 |

### 1.6.3 数据预处理流水线

```
PDF 文件
  │
  ├─→ PyPDFLoader → 全文文本
  │     │
  │     ├─→ split_into_chapters() → 17 个章节 Document
  │     │     │
  │     │     ├─→ create_chapter_summary() → 17 个摘要 Document
  │     │     │     │
  │     │     │     └─→ encode_chapter_summaries() → chapter_summaries_vector_store
  │     │     │
  │     │     └─→ (保留章节元数据)
  │     │
  │     └─→ extract_book_quotes_as_documents() → ~500 条引文 Document
  │           │
  │           └─→ encode_quotes() → book_quotes_vectorstore
  │
  └─→ RecursiveCharacterTextSplitter → ~700 个文本块
        │
        └─→ FAISS.from_documents() → chunks_vector_store
```

---

## 1.7 项目局限性与边界

### 1.7.1 已知局限

| 局限 | 影响 | 缓解措施 |
|------|------|----------|
| **单文档** | 仅支持一本书 | 可扩展为多库路由 |
| **英文为主** | 嵌入模型针对英文优化 | 可更换多语言嵌入 |
| **无持久化对话** | 每次独立执行 | 需增加会话存储 |
| **高 LLM 成本** | 每问题 ~10-30 次调用 | 可缓存、可蒸馏 |
| **无用户认证** | 任何人可访问 | 部署时增加认证 |
| **FAISS 单机** | 不支持分布式 | 可替换为 Qdrant/Milvus |

### 1.7.2 不适用场景

- ❌ **实时对话系统**: 延迟太高（30-120s）
- ❌ **大规模文档集**: FAISS 单机性能瓶颈
- ❌ **高并发场景**: 无异步架构
- ❌ **需要 100% 精确**: LLM 推理存在概率性错误

---

## 1.8 本章小结

本项目是一个**教学级/研究级的 Agentic RAG 参考实现**，核心价值在于：

1. **架构示范**: 展示了如何使用 LangGraph 构建复杂的多步推理 Agent
2. **质量闭环**: 检索-蒸馏-验证-幻觉检测的完整质量保障
3. **可视化**: 实时图执行可视化，极大提升可解释性
4. **可扩展**: 模块化设计，易于替换组件（LLM、向量存储、检索策略）

**下一章**: [02-c4-architecture.md](./02-c4-architecture.md) — 使用 C4 模型从四个抽象层级完整描述系统架构。

---

☕️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../res/promotion.jpg)