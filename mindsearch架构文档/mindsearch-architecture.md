# MindSearch 架构文档

> **项目**: [InternLM/MindSearch](https://github.com/InternLM/MindSearch)
> **论文**: [MindSearch: Mimicking Human Minds Elicits Deep AI Searcher](https://arxiv.org/abs/2407.20183)
> **许可**: Apache-2.0
> **作者**: Zehui Chen, Kuikun Liu, Qiuchen Wang, Jiangning Liu, Wenwei Zhang, Kai Chen, Feng Zhao (Shanghai AI Laboratory)
> **技术栈**: Python 3.10 / FastAPI / Lagent v0.5 / React 18 + Vite + Ant Design + ReactFlow
> **代码规模**: 24 个 Python 文件 / ~3,984 行后端代码 + 22 个 TS/TSX 文件前端

---

## 目录

1. [项目概述](#1-项目概述)
2. [架构总览](#2-架构总览)
3. [核心概念与数据模型](#3-核心概念与数据模型)
4. [核心模块详解](#4-核心模块详解)
5. [搜索图引擎：WebSearchGraph](#5-搜索图引擎websearchgraph)
6. [双层 Agent 系统](#6-双层-agent-系统)
7. [前端架构](#7-前端架构)
8. [API 与通信协议](#8-api-与通信协议)
9. [部署与运维](#9-部署与运维)
10. [架构亮点与改进建议](#10-架构亮点与改进建议)
11. [关键文件索引](#11-关键文件索引)

---

## 1. 项目概述

### 1.1 项目目标

MindSearch 是上海 AI Lab (Shanghai AI Laboratory) 开源的 **深度 AI 搜索引擎**，其核心创新是 **模仿人类思维模式** 进行网络搜索。与传统的单次搜索或简单的 ReAct 徏环不同，MindSearch 将用户的复杂问题分解为一个 **有向搜索图 (DAG)**，每个节点代表一个子问题，通过并行搜索和信息聚合来生成全面的最终回答。

**论文核心观点**：人类在搜索信息时并非线性地逐个查找，而是先将大问题拆解为子问题网络，再并行搜索无依赖的子问题，最后综合所有信息得出结论。MindSearch 通过 LLM + 代码执行实现了这一认知过程。

### 1.2 核心特性

| 特性 | 描述 |
|------|------|
| **思考图搜索** | 将问题分解为 DAG 搜索图，非线性的树状搜索策略 |
| **并行子问题搜索** | 无依赖的子问题节点并发执行，显著缩短延迟 |
| **双层 Agent 架构** | Planner Agent（规划器）+ Searcher Agent（搜索器）分工协作 |
| **流式 SSE 响应** | 全链路流式输出，前端实时展示搜索思考过程 |
| **多搜索引擎支持** | Bing / Google / DuckDuckGo / Brave / Tencent 五种搜索引擎 |
| **多 LLM 后端** | InternLM2.5 / GPT-4 / Qwen / SiliconFlow 等可插拔后端 |
| **中英双语** | 完整的中英文 prompt 和界面支持 |
| **可视化搜索图** | ReactFlow 渲染思维导图，展示搜索推理路径 |
| **三种前端** | React（完整可视化）/ Gradio / Streamlit（快速验证） |

### 1.3 代码统计

| 模块 | 文件数 | 行数 | 说明 |
|------|--------|------|------|
| `mindsearch/agent/` | 5 | ~930 | 核心 Agent 逻辑（planner + searcher + graph） |
| `mindsearch/` (顶层) | 3 | ~240 | FastAPI 服务、终端入口、`__init__` |
| `frontend/React/src/` | 22 | ~3,500+ | React 前端（含组件、样式） |
| `frontend/gradio_agentchatbot/` | 4 | ~800 | Gradio 自定义聊天组件 |
| `docker/msdl/` | 7 | ~500 | Docker 部署辅助工具 (MindSearch DownLoader) |
| **总计 Python** | **24** | **~3,984** | 后端代码 |

### 1.4 依赖清单

**Python 后端关键依赖**：

| 依赖 | 版本 | 用途 |
|------|------|------|
| `lagent` | 0.5.0rc2 | Agent 框架（InternLM 的轻量 Agent 库） |
| `fastapi` | - | Web 框架 |
| `uvicorn` | - | ASGI 服务器 |
| `sse-starlette` | - | SSE 流式响应 |
| `janus` | - | 同步/异步队列桥接 |
| `pydantic` | 2.6.4 | 数据校验 |
| `duckduckgo_search` | 5.3.1b1 | DuckDuckGo 搜索 |
| `gradio` | 5.7.1 | Gradio 前端 |
| `transformers` | 4.41.0 | HuggingFace 模型加载 |
| `tenacity` | - | 重试机制 |

**React 前端关键依赖**：

| 依赖 | 版本 | 用途 |
|------|------|------|
| `react` | 18.2.0 | UI 框架 |
| `antd` | 5.18.3 | 组件库 |
| `reactflow` | 11.11.3 | 搜索图可视化（DAG 节点渲染） |
| `@microsoft/fetch-event-source` | 2.0.1 | SSE 客户端 |
| `react-markdown` | 9.0.1 | Markdown 渲染 |
| `@antv/x6` | 2.18.1 | 图编辑引擎（思维导图） |
| `elkjs` | 0.9.3 | 图布局算法 |
| `axios` | 1.3.5 | HTTP 客户端 |
| `vite` | 4.2.1 | 构建工具 |

---

## 2. 架构总览

### 2.1 系统架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                         用户浏览器                                   │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  React 前端 (Vite + Ant Design + ReactFlow)                 │   │
│  │  ├── 搜索图可视化 (DAG 节点 + 边)                             │   │
│  │  ├── 聊天界面 (SSE 流式接收)                                  │   │
│  │  ├── 右侧面板 (节点详情：思考/搜索/阅读/结论)                  │   │
│  │  └── Markdown 渲染 + 引用标注 [[n]]                          │   │
│  └──────────────────────────┬──────────────────────────────────┘   │
└─────────────────────────────┼───────────────────────────────────────┘
                              │ POST /solve (SSE)
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    FastAPI 后端 (Port 8002)                         │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  app.py — SSE 端点 /solve                                    │   │
│  │  ├── sync mode: janus.Queue 桥接同步生成器 → 异步 SSE         │   │
│  │  └── async mode: 直接 async for → SSE                       │   │
│  └──────────────────────────┬──────────────────────────────────┘   │
│                              │                                       │
│  ┌──────────────────────────▼──────────────────────────────────┐   │
│  │  MindSearchAgent (Planner)                                   │   │
│  │  ├── 基于 StreamingAgentForInternLM                          │   │
│  │  ├── LLM 生成 Python 代码构建搜索图                            │   │
│  │  ├── output_format: InterpreterParser (代码解析)              │   │
│  │  └── ExecutionAction: exec() 执行 LLM 生成的代码              │   │
│  └──────────────────────────┬──────────────────────────────────┘   │
│                              │                                       │
│  ┌──────────────────────────▼──────────────────────────────────┐   │
│  │  WebSearchGraph (搜索图引擎)                                  │   │
│  │  ├── nodes: {root, node1, node2, ..., response}              │   │
│  │  ├── adjacency_list: DAG 邻接表                               │   │
│  │  ├── ThreadPoolExecutor (同步) / asyncio loop (异步)          │   │
│  │  └── searcher_resp_queue: 流式结果队列                         │   │
│  └──────────────────────────┬──────────────────────────────────┘   │
│                              │ 每个节点启动一个                       │
│  ┌──────────────────────────▼──────────────────────────────────┐   │
│  │  SearcherAgent × N (并行搜索器)                               │   │
│  │  ├── 基于 StreamingAgentForInternLM                          │   │
│  │  ├── output_format: PluginParser (工具调用解析)               │   │
│  │  ├── WebBrowser 工具: search() → browse() → 总结              │   │
│  │  └── 引用标注 [[n]] → URL 映射                                │   │
│  └──────────────────────────┬──────────────────────────────────┘   │
└─────────────────────────────┼───────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      外部服务                                       │
│  ┌────────────┐  ┌────────────┐  ┌────────────────────────────┐   │
│  │ LLM Server │  │ Search API │  │   Web Pages (URL 内容)      │   │
│  │ (InternLM/ │  │ (Bing/Goog-│  │   被 WebBrowser.browse()    │   │
│  │  GPT/Qwen) │  │  le/DDG/   │  │   抓取并返回摘要             │   │
│  │            │  │  Brave)    │  │                              │   │
│  └────────────┘  └────────────┘  └────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 分层架构

MindSearch 的架构可以清晰分为 **五层**：

**① 表示层 (Presentation Layer)** — React/Gradio/Streamlit 前端，负责用户交互、搜索图可视化和流式渲染

**② API 层 (API Layer)** — FastAPI + SSE，负责请求接收、异步桥接和流式推送

**③ 规划层 (Planning Layer)** — MindSearchAgent (Planner)，利用 LLM 将问题分解为搜索图代码

**④ 搜索层 (Search Layer)** — WebSearchGraph + SearcherAgent，执行图节点的并行搜索

**⑤ 工具层 (Tool Layer)** — Lagent WebBrowser，封装搜索引擎 API 和网页浏览

### 2.3 核心设计决策

**为什么不直接用 ReAct 循环？**

传统 ReAct（Reason+Act）是线性循环：思考 → 行动 → 观察 → 思考 → ...。对于复杂问题（如"比较 A、B、C 的优缺点"），线性搜索效率低下。MindSearch 的创新在于让 LLM 生成 **Python 代码** 来构建搜索图，利用代码的表达能力自然地描述并行和依赖关系。

**为什么用代码生成而不是 JSON 结构化输出？**

代码比 JSON 更灵活：
- 可以表达条件逻辑（`if` 某节点结果包含 X，则添加 Y 节点）
- 可以在同一个代码块中添加多个节点和边
- LLM 对 Python 代码的理解和生成能力已经非常成熟
- 通过 `exec()` 执行自然支持流式结果收集

---

## 3. 核心概念与数据模型

### 3.1 搜索图 (WebSearchGraph)

搜索图是 MindSearch 的核心数据结构，它是一个 **有向无环图 (DAG)**：

```
         ┌─────────┐
         │  root   │  ← 用户原始问题
         │ (用户问题)│
         └────┬────┘
          ┌───┴───┐
          ▼       ▼
    ┌──────────┐ ┌──────────┐
    │ node_1   │ │ node_2   │  ← 子问题（可并行搜索）
    │ (子问题A) │ │ (子问题B) │
    └─────┬────┘ └────┬─────┘
          │           │
          ▼           ▼
    ┌──────────┐ ┌──────────┐
    │ node_3   │ │ node_4   │  ← 基于前序结果的后续问题
    │ (子问题C) │ │ (子问题D) │
    └─────┬────┘ └────┬─────┘
          │           │
          └─────┬─────┘
                ▼
         ┌──────────┐
         │ response │  ← 最终回答节点（触发汇总）
         └──────────┘
```

**图的节点类型**：

| 类型 | 说明 | 字段 |
|------|------|------|
| `root` | 根节点，用户原始问题 | `content`, `type="root"` |
| `searcher` | 搜索节点，一个子问题 | `content`, `type="searcher"`, `response`, `memory`, `session_id` |
| `end` | 结束节点，触发最终汇总 | `type="end"` |

### 3.2 AgentMessage 消息流

MindSearch 基于 Lagent 的 `AgentMessage` schema 进行 Agent 间通信：

```python
AgentMessage(
    sender="user" | "MindSearchAgent" | "ActionExecutor" | "SearcherAgent",
    content=str | dict,           # 消息内容（文本或结构化数据）
    formatted=dict,               # 格式化数据（graph_state 等）
    stream_state=AgentStatusCode, # 流状态码
)
```

**AgentStatusCode 状态流转**：

```
SESSION_READY → STREAM_ING → CODING / PLUGIN_START → END
```

- `SESSION_READY`: Agent 会话就绪
- `STREAM_ING`: LLM 正在流式输出
- `CODING`: 代码生成中（Planner 生成 Python 代码）
- `PLUGIN_START`: 插件调用中（Searcher 调用搜索工具）
- `END`: 流结束

### 3.3 引用标注系统

MindSearch 的搜索结果使用 `[[n]]` 格式的引用标注，类似于学术引用：

- **生成端**：Searcher Agent 的 system prompt 要求每个关键点标注 `[[id]]`
- **收集端**：`_generate_references_from_graph()` 汇总所有节点的引用，重映射编号
- **展示端**：前端用正则 `[[\d+]]` 匹配并渲染为可点击的上标链接

引用与 URL 的映射关系存储在 `ref2url` 字典中：`{1: {"url": "...", "title": "..."}, ...}`

---

## 4. 核心模块详解

### 4.1 `mindsearch/agent/__init__.py` — Agent 工厂

**职责**：Agent 的创建和配置工厂，是整个系统的入口点。

**`init_agent()` 函数** 参数：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `lang` | `"cn"` | 语言（`cn` 中文 / `en` 英文） |
| `model_format` | `"internlm_server"` | LLM 后端类型 |
| `search_engine` | `"BingSearch"` | 搜索引擎 |
| `use_async` | `False` | 是否使用异步模式 |

**核心逻辑**：

1. **LLM 缓存**：使用全局 `LLM` 字典缓存已创建的 LLM 实例（按 `model_format` + `mode`），避免重复初始化
2. **异步转换**：如果 `use_async=True`，将 LLM 配置中的类型字符串改为 `lagent.llms.Async{ClassName}`
3. **WebBrowser 插件配置**：根据搜索引擎类型创建 `AsyncWebBrowser` 或 `WebBrowser`，`topk=6`（每次搜索返回 6 条结果）
4. **Agent 构建**：创建 `AsyncMindSearchAgent` 或 `MindSearchAgent`，传入：
   - `output_format=InterpreterParser`：解析 Planner 生成的 Python 代码
   - `searcher_cfg`：包含 Searcher 的 LLM、插件、prompt 模板
   - `summary_prompt`：最终汇总的 prompt
   - `max_turn=10`：最大对话轮次

**支持的 LLM 后端** (`mindsearch/agent/models.py`)：

| 标识符 | 类型 | 说明 |
|--------|------|------|
| `internlm_server` | `LMDeployServer` | InternLM2.5-7B 本地部署（LMDeploy） |
| `internlm_client` | `LMDeployClient` | InternLM2.5-7B 远程客户端 |
| `internlm_hf` | `HFTransformerCasualLM` | HuggingFace Transformers 加载 |
| `gpt4` | `GPTAPI` | OpenAI GPT-4 Turbo |
| `qwen` | `GPTAPI` | 阿里通义千问 (DashScope API) |
| `internlm_silicon` | `GPTAPI` | SiliconFlow 托管的 InternLM |

### 4.2 `mindsearch/app.py` — FastAPI 服务

**职责**：提供 SSE 流式 API 端点，是前后端通信的桥梁。

**API 设计**：

唯一端点 `POST /solve`，接收 `GenerationParams`：

```python
class GenerationParams(BaseModel):
    inputs: Union[str, List[Dict]]       # 用户查询
    session_id: int = random.randint()    # 会话 ID
    agent_cfg: Dict = dict()              # Agent 配置（保留）
```

**同步模式 (`run`)** vs **异步模式 (`run_async`)**：

| 方面 | 同步模式 | 异步模式 |
|------|----------|----------|
| Agent 类型 | `MindSearchAgent` | `AsyncMindSearchAgent` |
| 队列桥接 | `janus.Queue()` 同步→异步 | 直接 `async for` |
| 线程模型 | ThreadPoolExecutor 执行同步生成器 | 单线程 asyncio |
| 适用场景 | 兼容性好，调试方便 | 高并发，低延迟 |

**`_postprocess_agent_message()`** — SSE 消息后处理：

该函数是前后端数据契约的核心。它根据 `current_node` 的有无来区分两种状态：

- **有 current_node**（图执行阶段）：只返回当前节点的数据，过滤掉其他节点和敏感字段（`memory`、`session_id`、`ref2url`）
- **无 current_node**（Planner 思考阶段）：返回 Planner 的 thought 和 action

**客户端断连处理**：通过 `_request.is_disconnected()` 检测前端断开，及时停止生成。

**内存清理**：每次请求结束后 `memory_map.pop(session_id)` 清理会话内存。

### 4.3 `mindsearch/agent/streaming.py` — 流式 Agent 基类

**职责**：为 Lagent 的 Agent 添加流式输出能力。

**类继承体系**：

```
StreamingAgentMixin                    AsyncStreamingAgentMixin
       ↓                                       ↓
StreamingAgent (→ Agent)              AsyncStreamingAgent (→ AsyncAgent)
       ↓                                       ↓
StreamingAgentForInternLM             AsyncStreamingAgentForInternLM
(→ AgentForInternLM)                  (→ AsyncAgentForInternLM)
       ↓                                       ↓
MindSearchAgent                       AsyncMindSearchAgent
SearcherAgent                         AsyncSearcherAgent
```

**`StreamingAgentForInternLM.forward()`** 核心循环：

1. 调用内部 Agent (`StreamingAgent`) 进行 LLM 流式推理
2. 根据输出解析 `tool_type`（`plugin` 或 `interpreter`）
3. 如果有工具调用 → 执行工具 → 将结果作为新 message 继续
4. 如果没有工具调用 → 标记 `END`，结束循环
5. 最多循环 `max_turn` 次（默认 10）

### 4.4 `mindsearch/agent/mindsearch_prompt.py` — Prompt 工程

这是 MindSearch 最核心的知识产权之一。Prompt 质量直接决定搜索质量。

**三层 Prompt 体系**：

**① Planner Prompt (`GRAPH_PROMPT_CN/EN`)**：

角色设定为"可以利用 Jupyter 环境编程的程序员"，提供 `WebSearchGraph` 类的完整 API 文档（`add_root_node`、`add_node`、`add_response_node`、`add_edge`），并通过 few-shot 示例展示如何用 Python 代码构建搜索图。

关键约束（7 条注意事项）：
- 每个搜索节点必须是 **单一问题**，不允许复合问题
- 不要杜撰搜索结果，等待代码返回
- 不要重复提问
- `response` 节点必须单独添加
- 每次输出只包含一个代码块

**② Searcher Prompt (`searcher_system_prompt_cn/EN`)**：

角色设定为"可以调用网络搜索工具的智能助手"，要求：
- 每个关键点标注 `[[int]]` 引用
- 优先回答"当前问题"（相对于"主问题"）
- 工具调用格式：`<|action_start|><|plugin|>{"name": "...", "parameters": {...}}<|action_end|>`

**③ Summary Prompt (`FINAL_RESPONSE_CN/EN`)**：

指导 LLM 基于所有搜索节点的 Q&A 对，撰写一篇详细完备的最终回答，保持引用索引一致。

---

## 5. 搜索图引擎：WebSearchGraph

`mindsearch/agent/graph.py` 是 MindSearch 架构的核心创新所在。

### 5.1 WebSearchGraph 类

```python
class WebSearchGraph:
    is_async = False                     # 类级标志：是否异步模式
    SEARCHER_CONFIG = {}                 # 类级配置：Searcher Agent 的参数
    _SEARCHER_LOOP = []                  # asyncio 事件循环池（异步模式）
    _SEARCHER_THREAD = []                # 守护线程池（异步模式）

    def __init__(self):
        self.nodes: Dict[str, Dict]      # 节点字典
        self.adjacency_list: Dict[str, List]  # 邻接表
        self.future_to_query = dict()    # Future → 查询映射
        self.searcher_resp_queue = queue.Queue()  # 结果队列（流式核心）
        self.executor = ThreadPoolExecutor(max_workers=10)  # 同步线程池
        self.n_active_tasks = 0          # 活跃搜索任务计数
```

### 5.2 节点生命周期

**`add_root_node()`** — 创建根节点，存储用户原始问题。

**`add_node(node_name, node_content)`** — 添加搜索节点，核心逻辑：

1. 注册节点到 `self.nodes` 和 `self.adjacency_list`
2. **查找父节点**：遍历邻接表找到当前节点的所有前驱节点，收集它们的 Q&A 作为历史上下文
3. **启动 Searcher Agent**：
   - **异步模式**：`asyncio.run_coroutine_threadsafe()` 提交到随机选择的 event loop
   - **同步模式**：`ThreadPoolExecutor.submit()` 提交到线程池
4. Searcher Agent 的流式结果通过 `searcher_resp_queue` 传递给 `ExecutionAction`
5. `n_active_tasks += 1`

**`add_response_node()`** — 添加结束节点，标记搜索完成。

**`add_edge(start, end)`** — 添加有向边到邻接表，每条边包含 UUID 和状态码（1=进行中，2=未开始，3=已结束）。

### 5.3 ExecutionAction — 代码执行器

```python
class ExecutionAction(BaseAction):
    def run(self, command, local_dict, global_dict, stream_graph=False):
        # 1. 提取代码（从 LLM 输出中剥离 markdown 标记）
        command = extract_code(command)
        # 2. 执行代码（exec 在 global_dict 中已有 WebSearchGraph）
        exec(command, global_dict, local_dict)
        # 3. 从代码中提取 graph.node() 调用
        node_list = re.findall(r"graph\.node\((.*?)\)", command)
        graph = local_dict["graph"]
        # 4. 轮询 searcher_resp_queue 直到所有任务完成
        while graph.n_active_tasks:
            while not graph.searcher_resp_queue.empty():
                node_name, _, _ = graph.searcher_resp_queue.get(timeout=60)
                # ... yield 流式消息 ...
        # 5. 返回最终节点数据
        return res, graph.nodes, graph.adjacency_list
```

**设计要点**：

- `exec()` 在包含 `WebSearchGraph` 类的全局命名空间中执行，LLM 生成的代码可以直接实例化和操作图
- `searcher_resp_queue` 是 **生产者-消费者模型** 的核心：Searcher Agent 是生产者（put 结果），ExecutionAction 是消费者（get 结果并 yield）
- 超时机制：`queue.get(timeout=60)` 防止单个节点搜索卡死整个流程
- 流式输出：每收到一个节点的部分结果，就 yield 一条 `AgentMessage`，前端实时更新

### 5.4 异步事件循环池

`start_loop(n=32)` 类方法启动 32 个 asyncio event loop（每个在一个 daemon thread 中），用于异步模式的并行搜索：

```python
@classmethod
def start_loop(cls, n: int = 32):
    while len(cls._SEARCHER_THREAD) < n:
        def _start_loop():
            loop = asyncio.new_event_loop()
            asyncio.set_event_loop(loop)
            cls._SEARCHER_LOOP.append(loop)
            loop.run_forever()
        thread = Thread(target=_start_loop, daemon=True)
        thread.start()
        cls._SEARCHER_THREAD.append(thread)
```

**为什么 32 个 loop？** 每个搜索节点需要一个独立的 Searcher Agent 运行 LLM 推理。如果使用单个 event loop，所有搜索节点串行执行。32 个 loop 允许最多 32 个搜索节点真正并行执行（通过 `random.choice(self._SEARCHER_LOOP)` 随机分配）。

---

## 6. 双层 Agent 系统

### 6.1 Planner Agent (MindSearchAgent)

**`MindSearchAgent.forward()`** 主循环（`mindsearch_agent.py`）：

```
for _ in range(max_turn):                    # 最多 10 轮
    for message in self.agent(message):       # LLM 流式推理
        # 设置 stream_state (STREAM_ING / CODING / PLUGIN_START)
        yield message

    if not message.formatted["tool_type"]:    # 无工具调用 → 结束
        return

    # 执行 LLM 生成的代码（构建/执行搜索图）
    gen = self.action.run(message.content, ...)  # ExecutionAction
    for graph_exec in gen:
        yield graph_exec                      # 流式输出图状态

    # 汇总引用
    reference, references_url = _generate_references_from_graph(gen.ret[1])
    _graph_state.update(node=..., adjacency_list=..., ref2url=...)

    if "add_response_node" in message.content:  # 触发最终汇总
        for message in self.agent(summary_prompt):
            yield message                      # 流式输出最终回答
        return

    # 将搜索结果作为新输入，继续下一轮规划
    message = AgentMessage(sender="ActionExecutor", content=reference, ...)
    yield message
```

**Planner 的思考链**：

1. **第 1 轮**：分析用户问题 → 生成 Python 代码构建初始搜索图（root + 几个子问题节点 + 边）
2. **代码执行**：`ExecutionAction.run()` 执行代码，搜索图中的节点并行搜索
3. **第 2 轮**：Planner 看到搜索结果 → 可能添加更多子问题节点，或添加 `response` 节点
4. **最终轮**：当 Planner 输出包含 `add_response_node` 时，使用 `summary_prompt` 生成最终回答

### 6.2 Searcher Agent

**`SearcherAgent.forward(question, topic, history)`**：

```python
def forward(self, question, topic, history=None, session_id=0, **kwargs):
    message = [self.user_input_template.format(question=question, topic=topic)]
    if history and self.user_context_template:
        message = [self.user_context_template.format_map(item) for item in history] + message
    message = "\n".join(message)
    return super().forward(message, session_id=session_id, **kwargs)
```

**Searcher 的三步搜索流程**（通过 WebBrowser 插件）：

1. **搜索 (search)**：LLM 生成搜索关键词（可以是多个），调用 `FastWebBrowser.search(query_list)`
2. **选择 (select)**：LLM 浏览搜索结果摘要，选择最相关的网页 `FastWebBrowser.select(index_list)`
3. **总结**：LLM 基于选中的网页内容，撰写带有 `[[n]]` 引用的详细回答

**Searcher 的上下文构建**：

- `topic`：原始问题（主问题），帮助 Searcher 保持方向
- `question`：当前子问题
- `history`：父节点的 Q&A 对，提供已知信息避免重复搜索

### 6.3 引用重映射

`_update_ref()` 和 `_generate_references_from_graph()` 负责引用编号的全局重映射：

```python
def _generate_references_from_graph(graph):
    ptr = 0  # 全局引用计数器
    for name, data_item in graph.items():
        if name in ["root", "response"]:
            continue
        ref2url = json.loads(data_item["memory"]["agent.memory"][2]["content"])
        updated_ref, ref2url, added_ptr = _update_ref(
            data_item["response"]["content"], ref2url, ptr)
        ptr += added_ptr
        references.append(f'## {data_item["content"]}\n\n{updated_ref}')
        references_url.update(ref2url)
    return "\n\n".join(references), references_url
```

每个 Searcher 节点的引用编号从 `[[1]]` 开始，汇总时需要连续重编号。

---

## 7. 前端架构

### 7.1 技术选型

React 前端使用 **Vite + React 18 + Ant Design + ReactFlow** 技术栈，兼容到 IE 11 和 Chrome 52（`@vitejs/plugin-legacy`）。

### 7.2 核心组件树

```
App.tsx
└── routes/routes.tsx
    └── MindSearchCon (pages/mindsearch/index.tsx)     ← 主页面
        ├── SessionItem × N                             ← 历史对话列表
        │   ├── MindMap / MindMapItem (ReactFlow)       ← 搜索图可视化
        │   └── Answer                                  ← 最终回答展示
        │       └── CustomMarkdown                       ← Markdown + 引用渲染
        ├── ChatRight                                    ← 右侧节点详情面板
        │   ├── QueryItem                                ← 搜索关键词展示
        │   ├── SearchItem                               ← 搜索结果列表
        │   └── EmptyPlaceholder                         ← 空状态
        ├── Input                                        ← 输入框
        └── Notice                                       ← 公告栏
```

### 7.3 SSE 数据处理 (`index.tsx`)

前端通过 `@microsoft/fetch-event-source` 连接后端 `/solve` 端点：

```typescript
fetchEventSource('/solve', {
    method: 'POST',
    body: JSON.stringify({ inputs: question }),
    onmessage(ev) {
        const res = JSON.parse(ev.data);
        if (res?.response?.stream_state === 0) {
            // 搜索完成
            setChatIsOver(true);
        } else {
            formatData(res);  // 更新图状态
        }
    }
});
```

**`formatData()` 和 `stashNodeInfo()`** — 前端状态管理核心：

由于 SSE 每次只返回当前活跃节点的增量数据，前端需要将碎片化的消息拼接成完整的节点信息。`stashNodeInfo()` 使用 `localStorage` 缓存每个节点的中间状态：

- `thinkingData`：Step 1 思考（搜索前）
- `queries`：搜索关键词
- `searchList`：搜索结果
- `readingData`：Step 2 思考（阅读网页后）
- `selectedIds`：选中的网页 ID
- `conclusion`：节点结论

### 7.4 引用渲染 (`tools.ts`)

```typescript
// [[1]] → <i class='f custom'>1</i>
export const replaceStr = (str: string) => {
    return str.replace(/\[\[(\d+)\]\]/g, (match, number) => {
        return `<i class='f custom'>${number}</i>`;
    });
};

// [[1]][[2]] → <span class="mergeQuo" data-ids="1,2">...</span>
export const mergeReplaceToDiv = (str: string) => { ... };
```

### 7.5 Vite 代理配置

```typescript
server: {
    port: 8080,
    proxy: {
        "/solve": {
            target: "",          // 需配置为后端地址，如 http://127.0.0.1:8002
            changeOrigin: true,
        },
    },
}
```

---

## 8. API 与通信协议

### 8.1 唯一端点

```
POST /solve
Content-Type: application/json
Accept: text/event-stream (SSE)

Request Body:
{
    "inputs": "用户问题",
    "session_id": 123456
}

Response: SSE Stream
data: {"current_node": null, "response": {"sender": "...", "content": "...", "formatted": {...}}}
data: {"current_node": "node1", "response": {"formatted": {"node": {"node1": {...}}}}}
data: {"current_node": null, "response": {"stream_state": 0}}  ← 结束标志
```

### 8.2 SSE 消息类型

| 阶段 | `current_node` | `formatted` 关键字段 | 说明 |
|------|----------------|----------------------|------|
| Planner 思考 | `null` | `thought`, `action` | LLM 正在生成 Python 代码 |
| 图执行 | `"node_name"` | `node`, `adjacency_list` | 搜索图正在执行 |
| 最终回答 | `null` | `thought` | LLM 正在生成最终回答 |
| 结束 | `null` | `stream_state: 0` | 整个会话结束 |

### 8.3 Ping 机制

SSE 连接配置 `ping=300`（每 300 秒发送一次 ping），防止代理服务器超时断开长连接。

---

## 9. 部署与运维

### 9.1 三种部署方式

**① 直接运行**（开发模式）：

```bash
# 后端
python -m mindsearch.app --lang cn --model_format internlm_server \
    --search_engine DuckDuckGo --asy

# 前端
cd frontend/React && npm install && npm start
```

**② Docker 部署**：

```dockerfile
FROM continuumio/miniconda3
RUN conda create --name fastapi python=3.10 -y && \
    conda run -n fastapi pip install -r requirements.txt
EXPOSE 8000
CMD ["python3", "-m", "mindsearch.app", "--asy", "--host", "0.0.0.0", "--port", "8002"]
```

**③ MindSearch DownLoader (MSDL)** — `docker/msdl/` 目录：

一个交互式的 Docker 部署辅助工具（7 个 Python 文件），支持：
- 交互式配置（搜索引擎、API Key、模型选择）
- 自动生成 docker-compose.yml
- 多语言支持（i18n.py）
- 用户交互引导（user_interaction.py）

### 9.2 环境变量配置

```
# .env 文件
OPENAI_API_KEY=          # OpenAI API 密钥
OPENAI_API_BASE=         # 自定义 API 地址
OPENAI_MODEL=            # 模型名称
SILICON_API_KEY=         # SiliconFlow API 密钥
SILICON_MODEL=           # SiliconFlow 模型名称
WEB_SEARCH_API_KEY=      # 搜索引擎 API 密钥（Bing/Google/Brave）
TENCENT_SEARCH_SECRET_ID=    # 腾讯搜索 Secret ID
TENCENT_SEARCH_SECRET_KEY=   # 腾讯搜索 Secret Key
```

### 9.3 CLI 参数

```bash
python -m mindsearch.app [OPTIONS]

Options:
  --host TEXT              服务地址 (默认: 0.0.0.0)
  --port INTEGER           服务端口 (默认: 8002)
  --lang TEXT              语言 cn/en (默认: cn)
  --model_format TEXT      LLM 后端 (默认: internlm_server)
  --search_engine TEXT     搜索引擎 (默认: BingSearch)
  --asy                    启用异步模式
```

### 9.4 调试

```bash
# 终端调试模式（不需要前端）
python -m mindsearch.terminal

# 后端直连测试
python backend_example.py
```

---

## 10. 架构亮点与改进建议

### 10.1 架构亮点

**① 代码生成即搜索规划**

MindSearch 最创新的设计是让 LLM 生成 Python 代码来构建搜索图。这比 JSON/Function Calling 更灵活：
- 代码天然支持并行（多个 `add_node` 调用在同一代码块中）
- `exec()` 执行后自然产生副作用（节点和边的创建）
- LLM 对 Python 代码的生成和理解能力极强

**② 生产者-消费者流式架构**

`searcher_resp_queue` 精妙地将多个并行 Searcher Agent 的流式输出汇聚到单一流中：
- Searcher Agent 作为生产者，每次 LLM 输出一段文字就 `queue.put()`
- `ExecutionAction` 作为消费者，`queue.get()` 并 yield 到上层
- 前端通过 SSE 实时收到所有节点的增量更新

**③ 双层 Agent 分工明确**

- Planner（规划器）：全局视角，决定"搜索什么"和"如何连接"
- Searcher（搜索器）：局部视角，决定"如何搜索"和"如何总结"
- 两层 Agent 共享同一个 LLM 实例（通过 `searcher_cfg` 传递），减少资源消耗

**④ 前端可视化思维导图**

ReactFlow 渲染搜索图 DAG，用户可以：
- 实时看到问题被分解为子问题的过程
- 点击任意节点查看详细搜索过程（思考→搜索→阅读→结论）
- 理解 AI 的推理路径，增强可信度

### 10.2 技术挑战

**① `exec()` 安全风险**

`ExecutionAction.run()` 使用 `exec()` 执行 LLM 生成的代码，存在注入风险。在生产环境中应考虑沙箱化（如 RestrictedPython 或容器隔离）。

**② 无状态 API**

每次请求都会重新 `init_agent()`，没有会话持久化。`session_id` 仅用于内存中的消息映射，请求结束后清理。多轮对话不支持上下文延续。

**③ 全局状态泄漏**

`WebSearchGraph.SEARCHER_CONFIG` 和 `LLM` 是类级/全局变量，并发请求可能互相干扰。

**④ 前端 localStorage 依赖**

节点中间状态存储在 `localStorage.stashedNodes`，刷新页面会丢失数据。

### 10.3 改进建议

| 优先级 | 建议 | 说明 |
|--------|------|------|
| 高 | **沙箱化 exec()** | 使用 RestrictedPython 或 subprocess 隔离代码执行 |
| 高 | **会话持久化** | 添加 Redis/数据库存储对话历史 |
| 高 | **并发安全** | 将全局变量改为请求级实例 |
| 中 | **搜索结果缓存** | 相同查询的搜索结果缓存，减少 API 调用 |
| 中 | **动态 max_turn** | 根据问题复杂度调整最大轮次 |
| 中 | **流式断点续传** | SSE 断开后从上次位置继续 |
| 低 | **多模态搜索** | 支持图片/PDF 搜索 |
| 低 | **自定义搜索图** | 允许用户手动编辑/修剪搜索图 |

### 10.4 性能分析

**瓶颈分析**：

1. **LLM 推理延迟**：Planner 需要多轮推理（分解→搜索→再分解→汇总），每轮 LLM 调用延迟 1-5 秒
2. **搜索引擎 API**：每次搜索 0.5-2 秒，受限于搜索引擎速率限制
3. **网页内容抓取**：`WebBrowser.browse()` 需要下载和解析网页，延迟 1-3 秒

**优化措施**：
- 异步模式（`--asy`）允许搜索节点并行执行
- `topk=6` 限制每次搜索返回 6 条结果，平衡信息量和延迟
- `max_turn=10` 限制最大规划轮次，防止无限循环
- 32 个 asyncio event loop 支持高并发搜索

---

## 11. 关键文件索引

### 后端核心文件

| 文件 | 行数 | 职责 |
|------|------|------|
| `mindsearch/agent/__init__.py` | 82 | Agent 工厂（init_agent） |
| `mindsearch/agent/mindsearch_agent.py` | 210 | Planner Agent 主循环 |
| `mindsearch/agent/graph.py` | 307 | WebSearchGraph + ExecutionAction + SearcherAgent |
| `mindsearch/agent/streaming.py` | 203 | 流式 Agent 基类（StreamingAgentForInternLM） |
| `mindsearch/agent/mindsearch_prompt.py` | 326 | 所有 Prompt 模板（Planner/Searcher/Summary） |
| `mindsearch/agent/models.py` | 95 | LLM 后端配置（6 种模型） |
| `mindsearch/app.py` | 176 | FastAPI SSE 端点 |
| `mindsearch/terminal.py` | 66 | 终端调试入口 |
| `backend_example.py` | 37 | 后端直连测试脚本 |

### 前端核心文件

| 文件 | 职责 |
|------|------|
| `frontend/React/src/pages/mindsearch/index.tsx` | 主页面（SSE 接收 + 状态管理） |
| `frontend/React/src/pages/mindsearch/utils/tools.ts` | 引用渲染工具函数 |
| `frontend/React/src/pages/mindsearch/provider/context.tsx` | React Context |
| `frontend/React/src/pages/mindsearch/components/mind-map/` | ReactFlow 搜索图可视化 |
| `frontend/React/src/pages/mindsearch/components/answer/` | 最终回答 + Markdown 渲染 |
| `frontend/React/src/pages/mindsearch/components/chat-right/` | 右侧节点详情面板 |
| `frontend/React/src/pages/mindsearch/components/session-item/` | 单轮对话展示 |
| `frontend/React/vite.config.ts` | Vite 配置（代理 /solve） |

### 其他文件

| 文件 | 职责 |
|------|------|
| `docker/msdl/` | MindSearch DownLoader 部署工具（7 个 .py） |
| `frontend/mindsearch_gradio.py` | Gradio 前端 |
| `frontend/mindsearch_streamlit.py` | Streamlit 前端 |
| `frontend/gradio_agentchatbot/` | Gradio 自定义聊天组件（4 个 .py） |
| `Dockerfile` | Docker 部署配置 |
| `.env.example` | 环境变量模板 |
| `requirements.txt` | Python 依赖 |

---

## 附录：完整执行流程时序

```
用户输入: "嫦娥6号上有哪些国际科学载荷？"
    │
    ▼
[1] React 前端 POST /solve {"inputs": "嫦娥6号..."}
    │
    ▼
[2] FastAPI 创建 MindSearchAgent，启动 SSE 流
    │
    ▼
[3] Planner Agent 第 1 轮 LLM 推理
    │  生成代码:
    │  ```python
    │  graph = WebSearchGraph()
    │  graph.add_root_node("嫦娥6号上有哪些国际科学载荷？")
    │  graph.add_node("payloads", "嫦娥6号有哪些国际科学载荷？")
    │  graph.add_node("functions", "这些载荷的科学目标是什么？")
    │  graph.add_edge("root", "payloads")
    │  graph.add_edge("root", "functions")
    │  graph.add_edge("payloads", "functions")
    │  graph.node("payloads"), graph.node("functions")
    │  ```
    │
    ▼
[4] ExecutionAction.exec() 执行代码
    │  ├── 创建 root 节点
    │  ├── 创建 "payloads" 节点 → 启动 SearcherAgent（线程/协程）
    │  ├── 创建 "functions" 节点 → 启动 SearcherAgent（线程/协程）
    │  └── 轮询 searcher_resp_queue
    │
    ▼
[5] SearcherAgent × 2 并行执行
    │  └── 每个 Searcher:
    │      ├── LLM 生成搜索关键词 → WebBrowser.search()
    │      ├── LLM 选择网页 → WebBrowser.select()
    │      └── LLM 总结 → 带引用的详细回答
    │
    ▼
[6] 流式结果通过 queue → SSE → React 前端
    │  前端实时更新搜索图可视化
    │
    ▼
[7] Planner Agent 第 2 轮 LLM 推理
    │  看到搜索结果，判断信息是否充分
    │  生成代码: graph.add_response_node()
    │
    ▼
[8] Planner Agent 最终轮 LLM 推理
    │  使用 FINAL_RESPONSE prompt 汇总所有搜索结果
    │  生成带引用的最终回答
    │
    ▼
[9] 前端收到 stream_state=0，渲染最终回答 + 搜索图
```

---

*本文档基于 MindSearch 源代码深度分析生成，涵盖 24 个 Python 文件（~4K 行）和 22 个 TS/TSX 前端文件的完整代码走读。*

---

# ❤️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../res/promotion.jpg)