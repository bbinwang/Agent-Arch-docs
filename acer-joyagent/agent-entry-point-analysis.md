# Agent 入口调用分析

## 1. 应用启动入口

| 服务 | 文件 | 说明 |
|------|------|------|
| Spring Boot 后端 | `genie-backend/.../GenieApplication.java:9` | `main()` 启动 Spring Boot，端口 8080 |
| Python Tool 服务 | `genie-tool/server.py:74` | `uvicorn.run()` 启动 FastAPI，端口 1601 |
| Python MCP Client | `genie-client/server.py:121` | `uvicorn.run()` 启动 MCP 服务，端口 8188 |
| 启动初始化 | `genie-backend/.../DataAgentInitRunner.java:30` | `CommandLineRunner`，初始化 Qdrant 和 ES |

---

## 2. 核心 Agent 入口：`POST /AutoAgent`

**文件：** `genie-backend/.../controller/GenieController.java`

- **Line 106-154：** `@PostMapping("/AutoAgent")` 方法 `AutoAgent(@RequestBody AgentRequest request)`
- 流程：
  1. 创建 SSE emitter（line 113）
  2. 启动心跳（line 115）
  3. 构建 `AgentContext`（lines 124-138）
  4. 构建 `ToolCollection`（line 140）
  5. `AgentHandlerFactory.getHandler()` 选择 handler（line 142）
  6. `handler.handle(agentContext, request)` 执行 agent（line 144）

---

## 3. 调用链

### Flow 1：前端 → Multi-Agent（经 MultiAgentServiceImpl 代理）

```
UI (querySSE.ts)
  → POST /web/api/v1/gpt/queryAgentStreamIncr
    → GptProcessServiceImpl.queryMultiAgentIncrStream()
      → MultiAgentServiceImpl.searchForAgentRequest()
        → MultiAgentServiceImpl.handleMultiAgentRequest()
          → OkHttp POST http://127.0.0.1:8080/AutoAgent
            → GenieController.AutoAgent()
              → AgentHandlerFactory.getHandler() → handler.handle()
                → BaseAgent.run()（核心 Agent 循环）
```

**关键调用点：**

| 调用方 | 文件 | 行号 |
|--------|------|------|
| 前端 SSE 连接 | `ui/src/utils/querySSE.ts` | 4 |
| Controller 路由 | `GenieController.java` | 279 |
| GptProcessService | `GptProcessServiceImpl.java` | 28 |
| MultiAgentService | `MultiAgentServiceImpl.java` | 132 |
| Agent 入口 | `GenieController.java` | 106-154 |

### Flow 2：直接调用 `/AutoAgent`

```
外部 HTTP 客户端
  → POST /AutoAgent (GenieController:106)
    → AgentHandlerFactory.getHandler() → handler.handle()
      → BaseAgent.run()
```

---

## 4. Handler 分发（策略模式）

**接口：** `genie-backend/.../service/AgentHandlerService.java`
- `handle(AgentContext, AgentRequest)` — 执行方法
- `support(AgentContext, AgentRequest)` — 路由条件

**工厂：** `genie-backend/.../service/impl/AgentHandlerFactory.java`
- `getHandler()` 遍历注册的 handler，返回 `support()` 为 true 的那个

### 两个具体 Handler

| Handler | 文件 | 条件 | 行为 |
|---------|------|------|------|
| `ReactHandlerImpl` | `ReactHandlerImpl.java:20` | `deepThink == 0`（AgentType.REACT=5） | 创建 `ReactImplAgent` + `SummaryAgent`，运行 `executor.run(query)` |
| `PlanSolveHandlerImpl` | `PlanSolveHandlerImpl.java:30` | `deepThink == 1`（AgentType.PLAN_SOLVE=3） | 创建 `PlanningAgent` + `ExecutorAgent` + `SummaryAgent`，循环执行 plan→execute |

**Agent 类型枚举：** `genie-backend/.../agent/enums/AgentType.java`
- `COMPREHENSIVE(1)`, `WORKFLOW(2)`, `PLAN_SOLVE(3)`, `ROUTER(4)`, `REACT(5)`

---

## 5. Agent 类继承体系

```
BaseAgent (BaseAgent.java:30)
├── ReActAgent (ReActAgent.java:23)
│   ├── ReactImplAgent (ReactImplAgent.java:30)     ← ReactHandler 使用
│   ├── PlanningAgent (PlanningAgent.java:32)        ← PlanSolveHandler 使用
│   └── ExecutorAgent (ExecutorAgent.java:30)        ← PlanSolveHandler 使用
└── SummaryAgent (SummaryAgent.java:25)              ← 两者共用
```

**核心方法：**

| 类 | 方法 | 文件:行号 | 说明 |
|----|------|-----------|------|
| `BaseAgent` | `run(query)` | `BaseAgent.java:62-89` | **核心 Agent 循环**，迭代 `step()` 最多 `maxSteps` 次 |
| `ReActAgent` | `step()` | `ReActAgent.java:39-45` | 调用 `think()` → `act()` |
| `ReActAgent` | `think()` | `ReActAgent.java:28` | 推理步骤（抽象） |
| `ReActAgent` | `act()` | `ReActAgent.java:33` | 执行步骤（抽象） |

**调用关系：**

| Handler | 创建的 Agent | 调用位置 |
|---------|-------------|----------|
| `ReactHandlerImpl` | `ReactImplAgent` | `ReactHandlerImpl.java:33` `executor.run(query)` |
| `ReactHandlerImpl` | `SummaryAgent` | `ReactHandlerImpl.java:34` |
| `PlanSolveHandlerImpl` | `PlanningAgent` | `PlanSolveHandlerImpl.java:50,87` `planning.run()` |
| `PlanSolveHandlerImpl` | `ExecutorAgent` | `PlanSolveHandlerImpl.java:60,72` `executor.run()` |
| `PlanSolveHandlerImpl` | `SummaryAgent` | `PlanSolveHandlerImpl.java:90` |

---

## 6. Python 侧 Agent 入口

Java 后端通过 HTTP 调用 Python 工具服务（端口 1601）的 Agent：

| Agent | 入口方法 | 路由 | 路由注册位置 |
|-------|---------|------|-------------|
| `AutoAnalysisAgent` | `genie_tool/tool/auto_analysis.py:76` `run()` | `POST /auto_analysis` | `genie_tool/api/tool.py:354,368` |
| `NL2SQLAgent` | `genie_tool/tool/nl2sql.py:251` `run()` | `POST /nl2sql` | `genie_tool/api/tool.py:394,408` |
| `DeepSearch` | `genie_tool/tool/deepsearch.py:52` `run()` | `POST /deepsearch` | `genie_tool/api/tool.py:278` |
| `TableRAGAgent` | `genie_tool/tool/table_rag/table_rag.py:503` `run()` | `POST /table_rag` | `genie_tool/api/tool.py:317` |
| `CIAgent` | `genie_tool/tool/ci_agent.py:73` `_step_stream()` | 通过 `code_interpreter.py` 间接调用 | `code_interpreter.py:91,129` |

### Flow 3：Python Tool Agent 调用链

```
Java ExecutorAgent 工具调用（HTTP）
  → POST /auto_analysis | /nl2sql | /deepsearch | /table_rag
    → AutoAnalysisAgent.run() | NL2SQLAgent.run() | DeepSearch.run() | TableRAGAgent.run()
```
