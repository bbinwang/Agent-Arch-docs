# 第3章 系统流程与时序图

> **文件**: `docs/wangbin/03-flows-sequence.md`  
> **预计 Token**: ~20,000  
> **核心内容**: 10+ 核心业务流程，每个含 Mermaid 图 + 300-500 字解释

---

## 3.1 流程总览

GPT Researcher 的核心业务流程可以用以下总览图表示：

```mermaid
flowchart TD
    START([用户提交研究任务]) --> ROUTE{报告类型路由}
    
    ROUTE -->|research_report| BASIC[基础报告流程]
    ROUTE -->|detailed_report| DETAILED[详细报告流程]
    ROUTE -->|deep| DEEP[深度研究流程]
    ROUTE -->|multi_agents| MULTI[多Agent协作流程]
    ROUTE -->|subtopic_report| SUBTOPIC[子话题报告]
    
    BASIC -->|执行| RESEARCH[研究执行]
    DETAILED -->|分解| SUBTOPIC_GEN[子话题生成]
    DEEP -->|递归| DEEP_RECURSE[深度递归探索]
    MULTI -->|状态机| LANGGRAPH[LangGraph 编排]
    
    SUBTOPIC_GEN --> RESEARCH
    DEEP_RECURSE --> RESEARCH
    LANGGRAPH --> RESEARCH
    
    RESEARCH -->|收集| CONTEXT[上下文聚合]
    CONTEXT -->|生成| REPORT[报告撰写]
    REPORT -->|输出| OUTPUT[报告输出]
    
    OUTPUT -->|格式选择| FMT{输出格式}
    FMT -->|markdown| MD[Markdown]
    FMT -->|pdf| PDF[PDF 文档]
    FMT -->|docx| DOCX[Word 文档]
    
    MD --> END([返回用户])
    PDF --> END
    DOCX --> END
```

---

## 3.2 流程 1: 标准研究报告生成

### 3.2.1 时序图

```mermaid
sequenceDiagram
    participant User as 用户
    participant FE as Frontend
    participant API as FastAPI
    participant Agent as GPTResearcher
    participant Conductor as ResearchConductor
    participant Retriever as Retriever
    participant Scraper as Scraper
    participant LLM as LLM Provider
    participant Writer as ReportGenerator
    participant WS as WebSocket

    User->>FE: 输入研究查询
    FE->>API: POST /report/ (ResearchRequest)
    API->>Agent: GPTResearcher(query, report_type)
    
    Note over Agent: 初始化阶段
    Agent->>Agent: Config(config_path)
    Agent->>Agent: get_retrievers()
    Agent->>Agent: Memory(embedding_provider)
    Agent->>Agent: ResearchConductor(self)
    Agent->>Agent: ReportGenerator(self)
    
    Note over Agent: 研究阶段
    Agent->>Agent: conduct_research()
    
    alt 未指定 agent/role
        Agent->>LLM: choose_agent(query)
        LLM-->>Agent: (agent_name, role_prompt)
    end
    
    Agent->>Conductor: conduct_research()
    
    Note over Conductor: 查询规划
    Conductor->>Retriever: search(query)
    Retriever-->>Conductor: search_results
    Conductor->>LLM: plan_research_outline()
    LLM-->>Conductor: sub_queries[]
    
    Note over Conductor: 并行子查询
    par 并行执行子查询
        Conductor->>Retriever: search(sub_query_1)
        Retriever-->>Conductor: results_1
    and
        Conductor->>Retriever: search(sub_query_2)
        Retriever-->>Conductor: results_2
    and
        Conductor->>Retriever: search(sub_query_3)
        Retriever-->>Conductor: results_3
    end
    
    Note over Conductor: 网页抓取
    Conductor->>Scraper: scrape_urls(urls)
    par 并行抓取
        Scraper->>Scraper: scrape(url_1)
        Scraper->>Scraper: scrape(url_2)
    end
    Scraper-->>Conductor: scraped_content[]
    
    Note over Conductor: 上下文收集
    Conductor->>LLM: summarize_content()
    LLM-->>Conductor: summaries
    Conductor-->>Agent: context[]
    
    Note over Agent: 报告撰写阶段
    Agent->>Agent: write_report()
    Agent->>Writer: write_report(context)
    Writer->>LLM: generate_report_prompt()
    LLM-->>Writer: report_markdown
    Writer-->>Agent: report
    
    Note over Agent: 输出阶段
    Agent->>WS: stream_output(report)
    WS-->>FE: 实时推送
    Agent-->>API: report
    API-->>FE: JSON Response
    FE-->>User: 展示报告
```

### 3.2.2 详细解释 (450+ 字)

#### 流程步骤分解

**阶段 1: 初始化** (`agent.py:__init__`)
- 加载配置 (`Config`)
- 实例化检索器列表 (`get_retrievers`)
- 初始化嵌入模型 (`Memory`)
- 创建 Skill 组件 (ResearchConductor, ReportGenerator 等)
- 处理 MCP 配置（如果提供）

**阶段 2: Agent 选择** (`actions/agent_creator.py:choose_agent`)
- 如果未预定义 agent/role，调用 LLM 自动选择
- 发送 `auto_agent_instructions()` 提示词
- 解析 LLM 返回的 JSON，提取 `server` 和 `agent_role_prompt`
- 容错处理：JSON 解析失败时使用 `json_repair`，再失败使用正则提取，最后回退到默认 Agent

**阶段 3: 研究执行** (`skills/researcher.py:conduct_research`)
- 调用 `plan_research()` 生成子查询
- 对每个子查询执行搜索 (`get_search_results`)
- 收集所有搜索结果中的 URL
- 调用 `scrape_urls()` 并行抓取网页内容
- 对抓取的内容进行摘要和来源追踪
- 累积到 `self.context` 列表

**阶段 4: 报告撰写** (`skills/writer.py:write_report`)
- 检查上下文是否为空（空则返回拒绝消息）
- 调用 `generate_report()` 生成报告
- 使用 `smart_llm` 模型（默认 gpt-5.4）
- 流式传输到 WebSocket

**阶段 5: 输出**
- Markdown 格式返回
- 可选转换为 PDF/DOCX
- 通过 WebSocket 实时推送进度

#### 异常处理

| 异常场景 | 处理方式 | 代码位置 |
|---------|---------|---------|
| LLM 调用失败 | 最多 10 次重试 | `utils/llm.py:create_chat_completion` |
| Strategic LLM 失败 | 降级到 Smart LLM | `actions/query_processing.py:generate_sub_queries` |
| 搜索无结果 | 返回空列表，继续处理 | `retrievers/tavily/tavily_search.py` |
| 单 URL 抓取失败 | 跳过该 URL，不影响整体 | `scraper/scraper.py:extract_data_from_url` |
| JSON 解析失败 | json_repair → regex → 默认 | `actions/agent_creator.py:handle_json_error` |
| 上下文为空 | 返回拒绝消息 | `skills/writer.py:write_report` |

#### 性能特征

- **并行度**: 子查询并行搜索、URL 并行抓取
- **I/O 模型**: 全异步 asyncio
- **线程池**: 阻塞 HTTP 调用通过 `asyncio.to_thread()` 卸载
- **并发控制**: Semaphore 限制同时抓取数（默认 15）

---

## 3.3 流程 2: 深度研究 (Deep Research)

### 3.3.1 时序图

```mermaid
sequenceDiagram
    participant User as 用户
    participant Agent as GPTResearcher
    participant Deep as DeepResearchSkill
    participant LLM as LLM Provider
    participant Retriever as Retriever
    participant Scraper as Scraper
    participant Progress as ProgressTracker

    User->>Agent: conduct_research(on_progress)
    Agent->>Deep: _handle_deep_research(on_progress)
    
    Note over Deep: 初始化深度研究
    Deep->>Deep: 设置 breadth=3, depth=2, concurrency=4
    
    Note over Deep: 第 1 层递归 (depth=1)
    Deep->>LLM: generate_search_queries(query, num_queries=3)
    LLM-->>Deep: queries[{query, researchGoal}]
    
    par 并行执行 3 个查询
        Deep->>Retriever: search(query_1)
        Retriever-->>Deep: results_1
    and
        Deep->>Retriever: search(query_2)
        Retriever-->>Deep: results_2
    and
        Deep->>Retriever: search(query_3)
        Retriever-->>Deep: results_3
    end
    
    Deep->>Scraper: scrape_urls(all_urls)
    Scraper-->>Deep: scraped_content
    
    Deep->>LLM: analyze_results_and_generate_followups()
    LLM-->>Deep: {learnings[], followUpQuestions[]}
    
    Deep->>Progress: update(depth=1, queries=3)
    Progress-->>User: on_progress callback
    
    Note over Deep: 第 2 层递归 (depth=2)
    loop 对每个 followUpQuestion
        Deep->>LLM: generate_search_queries(followup)
        LLM-->>Deep: sub_queries
        
        par 并行子查询
            Deep->>Retriever: search(sub_query)
            Retriever-->>Deep: sub_results
        end
        
        Deep->>Scraper: scrape_urls(sub_urls)
        Scraper-->>Deep: sub_content
        
        Deep->>LLM: analyze_results()
        LLM-->>Deep: {learnings, new_followups}
    end
    
    Deep->>Progress: update(depth=2, queries=total)
    Progress-->>User: on_progress callback
    
    Note over Deep: 聚合所有发现
    Deep->>Deep: aggregate_learnings(all_learnings)
    Deep-->>Agent: context[]
    
    Agent->>Agent: write_report(context)
    Agent-->>User: final_report
```

### 3.3.2 详细解释 (450+ 字)

#### 深度研究算法

深度研究采用**递归探索**策略，核心参数：

| 参数 | 默认值 | 说明 |
|------|-------|------|
| `breadth` | 3 | 每层生成的查询数 |
| `depth` | 2 | 递归深度 |
| `concurrency` | 4 | 并发查询数 |

#### 递归树结构

```
                        Root Query
                       /     |     \
                  Query_1  Query_2  Query_3       (depth=1, breadth=3)
                  / | \    / | \    / | \
               Q1a Q1b Q1c ...                  (depth=2, breadth=3 each)
               
总查询数 = breadth^1 + breadth^2 + ... + breadth^depth
        = 3 + 9 = 12 (默认配置)
```

#### 关键函数解析

**1. `parse_search_queries_response()`** (`skills/deep_research.py`)
- 解析 LLM 返回的搜索查询
- 支持 JSON 和纯文本两种格式
- 使用 `json_repair` 容错解析
- 正则表达式作为最终回退

**2. `parse_research_results_response()`**
- 解析研究结果为 `learnings` 和 `followUpQuestions`
- 提取引用来源 (citations)
- 支持多种 JSON 格式

**3. `parse_follow_up_questions_response()`**
- 从 LLM 响应中提取后续问题
- 用于驱动下一层递归

#### 进度追踪

```python
@dataclass
class ResearchProgress:
    current_depth: int
    total_depth: int
    current_breadth: int
    total_breadth: int
    completed_queries: int
    total_queries: int
    current_query: str | None
```

进度通过 `on_progress` 回调实时推送到前端。

#### 上下文管理

- `MAX_CONTEXT_WORDS = 25000`: 上下文最大词数限制
- 超过限制时进行截断或压缩
- 使用 `visited_urls` 集合去重

---

## 3.4 流程 3: 详细报告 (Detailed Report)

### 3.4.1 时序图

```mermaid
sequenceDiagram
    participant User as 用户
    participant Detailed as DetailedReport
    participant MainAgent as GPTResearcher (Main)
    participant LLM as LLM Provider
    participant SubAgent1 as SubtopicResearcher_1
    participant SubAgent2 as SubtopicResearcher_2
    participant SubAgent3 as SubTopicResearcher_3

    User->>Detailed: DetailedReport(query, report_type="detailed")
    
    Note over Detailed: 阶段1: 初始研究
    Detailed->>MainAgent: conduct_research()
    MainAgent->>LLM: choose_agent()
    MainAgent->>LLM: plan_research()
    MainAgent->>MainAgent: 搜索 + 抓取
    MainAgent-->>Detailed: global_context
    
    Note over Detailed: 阶段2: 子话题生成
    Detailed->>MainAgent: get_subtopics()
    MainAgent->>LLM: generate_subtopics_prompt()
    LLM-->>MainAgent: subtopics[{task}]
    MainAgent-->>Detailed: subtopics
    
    Note over Detailed: 阶段3: 引言撰写
    Detailed->>MainAgent: write_introduction()
    MainAgent->>LLM: generate_report_introduction()
    LLM-->>MainAgent: introduction
    
    Note over Detailed: 阶段4: 子话题报告 (并行)
    par 子话题 1
        Detailed->>SubAgent1: GPTResearcher(subtopic_1, parent_query)
        SubAgent1->>SubAgent1: conduct_research()
        SubAgent1->>SubAgent1: write_report()
        SubAgent1-->>Detailed: report_1
    and 子话题 2
        Detailed->>SubAgent2: GPTResearcher(subtopic_2, parent_query)
        SubAgent2->>SubAgent2: conduct_research()
        SubAgent2->>SubAgent2: write_report()
        SubAgent2-->>Detailed: report_2
    and 子话题 3
        Detailed->>SubAgent3: GPTResearcher(subtopic_3, parent_query)
        SubAgent3->>SubAgent3: conduct_research()
        SubAgent3->>SubAgent3: write_report()
        SubAgent3-->>Detailed: report_3
    end
    
    Note over Detailed: 阶段5: 聚合报告
    Detailed->>Detailed: construct_detailed_report(intro, body)
    Detailed->>Detailed: add_references(visited_urls)
    Detailed-->>User: final_report
```

### 3.4.2 详细解释 (400+ 字)

#### 详细报告架构

详细报告采用**分治策略**：
1. 首先进行全局研究，获取整体上下文
2. 基于全局研究生成子话题列表
3. 为每个子话题创建独立的研究 Agent
4. 并行执行子话题研究
5. 聚合所有子话题报告为最终文档

#### 子话题研究特点

每个子话题研究器 (`GPTResearcher`) 具有以下特点：
- `report_type="subtopic_report"`: 标识为子话题报告
- `parent_query`: 继承父查询，提供上下文
- `visited_urls`: 共享已访问 URL 集合，避免重复抓取
- `agent/role`: 继承主 Agent 的角色设定

#### 上下文传递

```python
# 全局上下文传递给子话题
self.global_context = self.gpt_researcher.context
self.global_urls = self.gpt_researcher.visited_urls

# 子话题研究器接收
subtopic_assistant = GPTResearcher(
    query=current_subtopic_task,
    parent_query=self.query,
    visited_urls=self.global_urls,  # 共享 URL 去重
    agent=self.gpt_researcher.agent,  # 继承角色
)
```

#### 报告聚合

最终报告结构：
```
# 主标题 (来自 query)
## 引言 (write_introduction)
## 子话题 1 标题
子话题 1 报告内容
## 子话题 2 标题
子话题 2 报告内容
## 子话题 3 标题
子话题 3 报告内容
## 参考文献 (add_references)
```

---

## 3.5 流程 4: 多 Agent 协作 (Multi-Agent)

### 3.5.1 时序图

```mermaid
sequenceDiagram
    participant User as 用户
    participant API as FastAPI
    participant Chief as ChiefEditorAgent
    participant Graph as LangGraph StateGraph
    participant Research as ResearchAgent
    participant Editor as EditorAgent
    participant Writer as WriterAgent
    participant FactCheck as FactCheckerAgent
    participant Visualizer as VisualizerAgent
    participant Publisher as PublisherAgent
    participant Human as HumanAgent

    User->>API: POST /api/multi_agents
    API->>Chief: ChiefEditorAgent(task)
    
    Note over Chief: 初始化 Agent 团队
    Chief->>Chief: _initialize_agents()
    Chief->>Research: ResearchAgent()
    Chief->>Editor: EditorAgent()
    Chief->>Writer: WriterAgent()
    Chief->>FactCheck: FactCheckerAgent()
    Chief->>Visualizer: VisualizerAgent()
    Chief->>Publisher: PublisherAgent()
    Chief->>Human: HumanAgent()
    
    Note over Chief: 创建工作流
    Chief->>Graph: _create_workflow(agents)
    Chief->>Graph: compile()
    
    Note over Graph: 执行状态机
    Graph->>Research: run_initial_research()
    Research-->>Graph: initial_data
    
    Graph->>Editor: plan_research(initial_data)
    Editor-->>Graph: research_plan
    
    Graph->>Human: review_plan(plan)
    Human-->>Graph: human_feedback
    
    alt feedback == None (接受)
        Graph->>Research: run_parallel_research(plan)
        Research-->>Graph: research_data
    else feedback == 修订
        Graph->>Editor: plan_research(revised)
        Note over Editor: 循环直到接受或达到最大修订次数
    end
    
    Graph->>Writer: run(research_data)
    Writer-->>Graph: draft_report
    
    Graph->>FactCheck: run(draft_report)
    FactCheck-->>Graph: fact_check_result
    
    alt 事实检查通过
        Graph->>Visualizer: run(verified_report)
        Visualizer-->>Graph: visual_report
    else 需要修订
        Graph->>Writer: run(revised_feedback)
        Note over Writer: 循环直到通过
    end
    
    Graph->>Publisher: run(visual_report)
    Publisher-->>Graph: final_output
    
    Graph-->>Chief: result
    Chief-->>API: research_report
    API-->>User: 报告
```

### 3.5.2 详细解释 (450+ 字)

#### LangGraph 状态机设计

多 Agent 协作基于 LangGraph 的 `StateGraph`，定义了以下状态：

```python
class ResearchState(TypedDict):
    task: Dict                    # 任务信息
    initial_data: str             # 初始研究结果
    research_plan: str            # 研究计划
    human_feedback: str | None    # 人类反馈
    revisions_count: int          # 修订计数
    research_data: str            # 研究数据
    draft_report: str             # 草稿报告
    fact_check_result: str        # 事实检查结果
    visual_report: str            # 可视化报告
    final_output: str             # 最终输出
```

#### 状态转换图

```mermaid
stateDiagram-v2
    [*] --> browser
    browser --> planner
    planner --> human
    human --> researcher : accept
    human --> planner : revise
    human --> researcher : force_accept
    researcher --> writer
    writer --> fact_checker
    fact_checker --> visualizer : accept
    fact_checker --> writer : revise
    visualizer --> publisher
    publisher --> [*]
```

#### Agent 角色定义

| Agent | 职责 | 输入 | 输出 |
|-------|------|------|------|
| ResearchAgent | 初始研究 + 并行深入研究 | task/plan | research_data |
| EditorAgent | 研究计划制定 | initial_data | research_plan |
| WriterAgent | 报告撰写 | research_data | draft_report |
| FactCheckerAgent | 事实核查 | draft_report | verified/revised |
| VisualizerAgent | 可视化增强 | verified_report | visual_report |
| PublisherAgent | 最终发布 | visual_report | final_output |
| HumanAgent | 人类审核 | plan | feedback |

#### Human-in-the-Loop

```python
# 条件边：人类反馈路由
workflow.add_conditional_edges(
    'human',
    lambda state: (
        "accept" if state['human_feedback'] is None
        else "force_accept" if state.get('revisions_count', 0) >= MAX_REVISIONS
        else "revise"
    ),
    {"accept": "researcher", "force_accept": "researcher", "revise": "planner"}
)
```

- `human_feedback is None`: 人类接受计划
- `revisions_count >= MAX_REVISIONS (5)`: 强制接受
- 否则: 返回 planner 修订

#### 事实核查循环

```python
workflow.add_conditional_edges(
    'fact_checker',
    self._route_fact_check,
    {"accept": "visualizer", "revise": "writer"}
)
```

事实检查不通过时返回 writer 修订，通过时进入 visualizer。

---

## 3.6 流程 5: MCP 两阶段检索

### 3.6.1 时序图

```mermaid
sequenceDiagram
    participant Agent as GPTResearcher
    participant MCPRet as MCPRetriever
    participant ClientMgr as MCPClientManager
    participant MCPServer as MCP Server
    participant ToolSel as MCPToolSelector
    participant MCPRes as MCPResearchSkill

    Agent->>MCPRet: MCPRetriever(query, researcher)
    
    Note over MCPRet: 初始化
    MCPRet->>MCPRet: _get_mcp_configs()
    MCPRet->>MCPRet: _get_config()
    MCPRet->>ClientMgr: MCPClientManager(mcp_configs)
    MCPRet->>ToolSel: MCPToolSelector(cfg, researcher)
    MCPRet->>MCPRes: MCPResearchSkill(cfg, researcher)
    
    Note over MCPRet: 阶段1: 获取工具列表
    MCPRet->>ClientMgr: get_or_create_client()
    ClientMgr->>MCPServer: connect (stdio/ws/http)
    MCPServer-->>ClientMgr: connection
    ClientMgr->>MCPServer: list_tools()
    MCPServer-->>ClientMgr: tools[{name, description, schema}]
    ClientMgr-->>MCPRet: all_tools
    
    Note over MCPRet: 阶段2: LLM 工具选择
    MCPRet->>ToolSel: select_tools(query, all_tools)
    ToolSel->>LLM: generate_mcp_tool_selection_prompt()
    LLM-->>ToolSel: selected_tools[{index, name, score}]
    ToolSel-->>MCPRet: selected_tools
    
    Note over MCPRet: 阶段3: 研究执行
    MCPRet->>MCPRes: execute_research(query, selected_tools)
    MCPRes->>LLM: generate_mcp_research_prompt()
    
    LLM->>MCPServer: call_tool(tool_name, args)
    MCPServer-->>LLM: tool_result
    
    LLM->>MCPServer: call_tool(another_tool, args)
    MCPServer-->>LLM: tool_result
    
    LLM-->>MCPRes: research_findings
    MCPRes-->>MCPRet: results
    
    MCPRet-->>Agent: search_results
```

### 3.6.2 详细解释 (400+ 字)

#### MCP 两阶段方法

**阶段 1: 工具选择 (Tool Selection)**
- 从所有连接的 MCP 服务器获取可用工具列表
- 使用 LLM 分析查询与工具的相关性
- 选择 2-3 个最相关的工具
- 返回工具名称、相关性评分、选择理由

**阶段 2: 研究执行 (Research Execution)**
- 将选中的工具绑定到 LLM
- LLM 自主决定何时调用工具、调用哪些工具
- 工具结果反馈给 LLM 进行综合分析
- 返回研究发现

#### MCP 传输协议支持

| 传输类型 | 说明 | 配置方式 |
|---------|------|---------|
| **stdio** | 标准输入/输出 | `command` + `args` |
| **websocket** | WebSocket 连接 | `connection_url: wss://...` |
| **streamable_http** | HTTP 流式 | `connection_url: https://...` |

#### 配置示例

```python
mcp_configs=[{
    "name": "my_search",
    "command": "python",
    "args": ["my_mcp_server.py"],
    "env": {"API_KEY": "xxx"}
}]

# 或远程连接
mcp_configs=[{
    "name": "remote_tool",
    "connection_url": "wss://mcp.example.com/ws",
    "connection_token": "Bearer xxx"
}]
```

#### MCP 策略

| 策略 | 说明 | 适用场景 |
|------|------|---------|
| `fast` | 仅对原始查询执行一次 MCP | 快速研究 |
| `deep` | 对所有子查询执行 MCP | 深度研究 |
| `disabled` | 跳过 MCP，使用 Web 检索器 | 无 MCP 配置 |

---

## 3.7 流程 6: WebSocket 实时流式传输

### 3.7.1 时序图

```mermaid
sequenceDiagram
    participant User as 用户浏览器
    participant WS as WebSocket
    participant Manager as WebSocketManager
    participant Agent as GPTResearcher
    participant Skills as Skills
    participant Queue as MessageQueue
    participant Sender as SenderTask

    User->>WS: new WebSocket("/ws")
    WS->>Manager: connect(websocket)
    Manager->>Manager: accept()
    Manager->>Queue: create Queue()
    Manager->>Sender: create_task(start_sender)
    
    User->>WS: start_research(task)
    WS->>Manager: start_streaming(task, ...)
    Manager->>Agent: run_agent(task, ...)
    
    Note over Agent: 研究执行中
    Agent->>Skills: conduct_research()
    
    Skills->>Agent: stream_output("logs", "starting_research", ...)
    Agent->>Manager: send_json({type, content, output})
    Manager->>Queue: put(message)
    Sender->>Queue: get()
    Sender->>WS: send_text(message)
    WS-->>User: 实时日志
    
    Skills->>Agent: stream_output("logs", "scraping_urls", ...)
    Agent->>Manager: send_json(...)
    Manager->>Queue: put(...)
    Sender->>WS: send_text(...)
    WS-->>User: 抓取进度
    
    Skills->>Agent: stream_output("images", "selected_images", ...)
    Agent->>Manager: send_json(..., data=images)
    Manager->>Queue: put(...)
    Sender->>WS: send_text(...)
    WS-->>User: 图像数据
    
    Agent->>Agent: write_report()
    Agent->>Skills: stream_output("logs", "writing_report", ...)
    Manager->>Queue: put(...)
    Sender->>WS: send_text(...)
    WS-->>User: 撰写进度
    
    Agent-->>Manager: report
    Manager-->>WS: final report
    WS-->>User: 完整报告
    
    Note over User: 断开连接
    User->>WS: disconnect
    WS->>Manager: disconnect(websocket)
    Manager->>Sender: cancel()
    Manager->>WS: close()
```

### 3.7.2 详细解释 (350+ 字)

#### WebSocket 架构

WebSocket 连接由 `WebSocketManager` 管理，每个连接包含：
- `active_connections`: 活跃连接列表
- `message_queues`: 每个连接的消息队列
- `sender_tasks`: 每个连接的发送任务

#### 消息类型

| 类型 | 说明 | 数据 |
|------|------|------|
| `logs` | 研究日志 | 步骤描述文本 |
| `images` | 图像数据 | 图像 URL 列表 |
| `cost` | 成本更新 | token 用量和费用 |
| `research_report` | 最终报告 | 完整报告文本 |

#### 消息队列模式

```python
# 生产者 (Agent)
await websocket.send_json({
    "type": "logs",
    "content": "scraping_urls",
    "output": "🌐 Scraping content from 5 URLs...",
    "metadata": None
})

# 消费者 (Sender Task)
while True:
    message = await queue.get()
    if message is None:  # Shutdown signal
        break
    await websocket.send_text(message)
```

#### 连接生命周期

1. **连接**: `connect()` → accept → 创建 Queue → 启动 Sender Task
2. **通信**: Agent 通过 Queue 推送消息，Sender Task 转发到 WebSocket
3. **断开**: `disconnect()` → 取消 Sender Task → 清理 Queue → 关闭连接

---

## 3.8 流程 7: 图像生成与嵌入

### 3.8.1 流程图

```mermaid
flowchart TD
    START[研究上下文] --> PLAN[分析可视化机会]
    PLAN -->|LLM 分析| CONCEPTS[识别 2-3 个概念]
    
    CONCEPTS --> GEN[并行生成图像]
    
    GEN -->|概念1| IMG1[生成图像 1]
    GEN -->|概念2| IMG2[生成图像 2]
    GEN -->|概念3| IMG3[生成图像 3]
    
    IMG1 --> CHECK1{生成成功?}
    IMG2 --> CHECK2{生成成功?}
    IMG3 --> CHECK3{生成成功?}
    
    CHECK1 -->|是| STORE1[存储图像信息]
    CHECK1 -->|否| SKIP1[跳过]
    CHECK2 -->|是| STORE2[存储图像信息]
    CHECK2 -->|否| SKIP2[跳过]
    CHECK3 -->|是| STORE3[存储图像信息]
    CHECK3 -->|否| SKIP3[跳过]
    
    STORE1 --> AGG[聚合图像列表]
    STORE2 --> AGG
    STORE3 --> AGG
    SKIP1 --> AGG
    SKIP2 --> AGG
    SKIP3 --> AGG
    
    AGG --> EMBED[嵌入报告]
    EMBED --> END[最终报告]
```

### 3.8.2 详细解释 (350+ 字)

#### 图像生成流程

**1. 图像规划** (`ImageGenerator.plan_and_generate_images`)
- 分析研究上下文，识别可视化机会
- 使用 LLM 确定 2-3 个最适合插图的概念
- 为每个概念生成图像提示词

**2. 图像生成**
- 支持两种提供商：Google Gemini 和 ModelsLab
- 并行生成所有图像
- 可配置图像风格 (dark/light/auto)

**3. 图像嵌入**
- 预生成的图像在报告撰写前完成
- 图像信息传递给 `write_report()`
- 报告中嵌入图像引用

#### 配置选项

```python
IMAGE_GENERATION_ENABLED = False  # 总开关
IMAGE_GENERATION_PROVIDER = "google"  # google/modelslab
IMAGE_GENERATION_MODEL = "models/gemini-2.5-flash-image"
IMAGE_GENERATION_MAX_IMAGES = 3
IMAGE_GENERATION_STYLE = "dark"
```

---

## 3.9 流程 8: 聊天问答 (Chat)

### 3.9.1 时序图

```mermaid
sequenceDiagram
    participant User as 用户
    participant FE as Frontend
    participant API as FastAPI
    participant Chat as ChatAgentWithMemory
    participant Tavily as Tavily Client
    participant LLM as LLM Provider

    User->>FE: 输入问题
    FE->>API: POST /api/chat {report, messages}
    
    Note over API: 创建聊天 Agent
    API->>Chat: ChatAgentWithMemory(report, config)
    Chat->>Chat: 初始化 Tavily 客户端
    
    API->>Chat: chat(messages, None)
    
    Note over Chat: 工具调用判断
    Chat->>LLM: create_chat_completion_with_tools(messages, [search_tool])
    
    alt LLM 决定搜索
        LLM->>Chat: tool_call(search_tool, query)
        Chat->>Tavily: quick_search(query)
        Tavily-->>Chat: search_results
        Chat->>LLM: tool_result(results)
        LLM-->>Chat: final_response
    else LLM 直接回答
        LLM-->>Chat: response
    end
    
    Chat-->>API: (response_content, tool_calls_metadata)
    API-->>FE: {response: {role, content, timestamp, metadata}}
    FE-->>User: 展示回答
```

### 3.9.2 详细解释 (300+ 字)

#### 聊天 Agent 设计

`ChatAgentWithMemory` 是基于研究报告的交互式问答系统：
- 接收完整报告作为上下文
- 支持工具调用（网络搜索）
- 返回结构化响应（含工具调用元数据）

#### 工具调用

聊天 Agent 使用 `create_chat_completion_with_tools()` 实现工具调用：
- 定义 `search_tool` 工具
- LLM 自主决定是否需要搜索
- 搜索结果作为工具返回值反馈给 LLM

#### 响应格式

```json
{
  "response": {
    "role": "assistant",
    "content": "回答内容...",
    "timestamp": 1721234567890,
    "metadata": {
      "tool_calls": [
        {
          "tool": "quick_search",
          "args": {"query": "..."},
          "result": "..."
        }
      ]
    }
  }
}
```

---

## 3.10 流程 9: 爬虫调度与限速

### 3.10.1 流程图

```mermaid
flowchart TD
    START[URL 列表] --> DEDUP[URL 去重]
    DEDUP --> POOL[WorkerPool]
    
    POOL -->|获取 Worker| SEM{Semaphore<br/>可用?}
    
    SEM -->|否| WAIT[等待]
    WAIT --> SEM
    
    SEM -->|是| RATE{全局限速<br/>检查}
    
    RATE -->|需等待| SLEEP[等待间隔]
    SLEEP --> RATE
    
    RATE -->|可执行| DISPATCH[分配 URL]
    
    DISPATCH -->|PDF| PYMUPDF[PyMuPDFScraper]
    DISPATCH -->|ArXiv| ARXIV[ArxivScraper]
    DISPATCH -->|其他| SELECT{爬虫选择}
    
    SELECT -->|bs| BS[BeautifulSoup]
    SELECT -->|browser| SELENIUM[Selenium]
    SELECT -->|firecrawl| FC[FireCrawl]
    SELECT -->|nodriver| ND[NoDriver]
    
    PYMUPDF --> RESULT{抓取成功?}
    ARXIV --> RESULT
    BS --> RESULT
    SELENIUM --> RESULT
    FC --> RESULT
    ND --> RESULT
    
    RESULT -->|是| CONTENT[提取内容]
    RESULT -->|否| ERROR[记录错误]
    
    CONTENT --> LEN{内容>100字符?}
    LEN -->|是| RETURN[返回结果]
    LEN -->|否| SKIP[跳过]
    
    ERROR --> NEXT[下一个 URL]
    RETURN --> NEXT
    SKIP --> NEXT
    
    NEXT -->|还有 URL| SEM
    NEXT -->|完成| AGG[聚合结果]
    AGG --> END[返回内容+图像]
```

### 3.10.2 详细解释 (350+ 字)

#### WorkerPool 并发控制

```python
class WorkerPool:
    def __init__(self, max_workers: int, rate_limit_delay: float):
        self.executor = ThreadPoolExecutor(max_workers=max_workers)
        self.semaphore = asyncio.Semaphore(max_workers)
```

- `Semaphore`: 限制并发抓取数（默认 15）
- `ThreadPoolExecutor`: 执行阻塞 I/O 操作
- `GlobalRateLimiter`: 全局限速（跨所有 WorkerPool）

#### 爬虫选择逻辑

```python
def get_scraper(self, link):
    path = urlparse(link).path
    if path.lower().endswith(".pdf"):
        return PyMuPDFScraper
    elif "arxiv.org" in link:
        return ArxivScraper
    else:
        return SCRAPER_CLASSES[self.scraper]  # 默认爬虫
```

#### 限速机制

```python
# 全局限速器（单例）
class GlobalRateLimiter:
    async def wait_if_needed(self):
        async with self.lock:
            time_since_last = time.time() - self.last_request_time
            if time_since_last < self.rate_limit_delay:
                await asyncio.sleep(self.rate_limit_delay - time_since_last)
            self.last_request_time = time.time()
```

---

## 3.11 流程 10: 配置加载与 Retriever 工厂

### 3.11.1 时序图

```mermaid
sequenceDiagram
    participant Agent as GPTResearcher
    participant Config as Config
    participant Env as 环境变量
    participant RetFactory as get_retrievers()
    participant Retriever as Retriever Classes

    Agent->>Config: Config(config_path)
    
    Note over Config: 加载配置
    Config->>Config: load_config(config_path)
    Config->>Config: _set_attributes(config)
    
    Note over Config: 环境变量覆盖
    loop 每个配置项
        Config->>Env: os.getenv(key)
        Env-->>Config: env_value
        alt env_value 存在
            Config->>Config: convert_env_value()
        end
    end
    
    Note over Config: 解析 LLM 配置
    Config->>Config: parse_llm(fast_llm)
    Config->>Config: parse_llm(smart_llm)
    Config->>Config: parse_llm(strategic_llm)
    
    Note over Config: 解析嵌入配置
    Config->>Config: parse_embedding(embedding)
    
    Note over Config: 解析检索器
    Config->>Config: parse_retrievers(retriever_env)
    
    Agent->>Agent: get_retrievers(headers, cfg)
    
    Note over Agent: 确定检索器列表
    Agent->>RetFactory: headers.get("retrievers")
    alt headers 指定
        RetFactory-->>Agent: retrievers from headers
    else cfg 指定
        RetFactory-->>Agent: retrievers from cfg
    else 默认
        RetFactory-->>Agent: [TavilySearch]
    end
    
    Note over Agent: 实例化检索器类
    loop 每个检索器名称
        Agent->>Retriever: get_retriever(name)
        Retriever-->>Agent: RetrieverClass
    end
    
    Agent-->>Agent: retrievers: List[Class]
```

### 3.11.2 详细解释 (300+ 字)

#### 配置优先级

1. **环境变量** (最高优先级): `os.getenv(key)`
2. **配置文件**: JSON 文件中的值
3. **默认值** (`variables/default.py`): 硬编码默认值

#### LLM 配置格式

```python
# 格式: "provider:model"
"FAST_LLM": "openai:gpt-5.4-mini"
"SMART_LLM": "openai:gpt-5.4"
"STRATEGIC_LLM": "openai:gpt-5.4"

# parse_llm() 解析为:
# ("openai", "gpt-5.4-mini")
```

#### Retriever 解析

```python
# 支持逗号分隔的多检索器
"RETRIEVER": "tavily,exa,duckduckgo"

# parse_retrievers() 解析为:
# ["tavily", "exa", "duckduckgo"]
```

---

## 3.12 流程总结

| 流程 | 触发入口 | 核心组件 | 输出 |
|------|---------|---------|------|
| 标准研究 | `POST /report/` | BasicReport | Markdown 报告 |
| 深度研究 | `report_type="deep"` | DeepResearchSkill | 深度报告 |
| 详细报告 | `report_type="detailed"` | DetailedReport | 多章节报告 |
| 多 Agent | `POST /api/multi_agents` | LangGraph StateGraph | 协作报告 |
| MCP 检索 | `mcp_configs` 配置 | MCPRetriever | 工具增强结果 |
| WebSocket | `WebSocket /ws` | WebSocketManager | 实时流 |
| 图像生成 | `IMAGE_GENERATION_ENABLED` | ImageGenerator | 报告插图 |
| 聊天问答 | `POST /api/chat` | ChatAgentWithMemory | 交互回答 |
| 爬虫调度 | `scrape_urls()` | WorkerPool + Scraper | 网页内容 |
| 配置加载 | `Config.__init__` | Config + get_retrievers | 系统配置 |

---

> **下一章**: → `04-module-structure.md` — 模块/包结构与依赖分析

---

☕️ 制作不易，请我喝咖啡☕️关注我➕