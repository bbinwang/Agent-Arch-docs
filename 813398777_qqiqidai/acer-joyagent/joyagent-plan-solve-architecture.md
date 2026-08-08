# JoyAgent Plan-Solve 架构详解

> 基于 `joyagent-jdgenie/genie-backend` 源码分析
> 分析时间：2026-05-04

---

## 1. 整体架构概览

JoyAgent 采用 **Plan-Solve** 多 Agent 协作架构，核心思想是将复杂任务拆分为「规划」和「执行」两个阶段，通过 **PlanningAgent → ExecutorAgent** 的循环迭代完成任务。

```
┌─────────────────────────────────────────────────────────┐
│                    GenieController                       │
│                   (HTTP/SSE 入口)                        │
│                         │                               │
│                    AgentRequest                          │
│                         ▼                               │
│              ┌──────────────────────┐                   │
│              │  AgentHandlerFactory  │ ←── 策略模式路由   │
│              └──────────┬───────────┘                   │
│                         │                               │
│         ┌───────────────┼───────────────┐               │
│         ▼               ▼               ▼               │
│  PlanSolveHandler  ReactHandler    (可扩展...)           │
│         │                                               │
│         ▼                                               │
│  ┌─────────────────────────────────────────────┐        │
│  │          Plan-Solve 编排主循环                │        │
│  │                                             │        │
│  │  PlanningAgent → ExecutorAgent → Planning   │        │
│  │       ↓              ↓            Agent     │        │
│  │  PlanningTool    业务工具集        ↓         │        │
│  │                             SummaryAgent    │        │
│  └─────────────────────────────────────────────┘        │
│                         │                               │
│                    SSEPrinter                           │
│                    (流式输出)                             │
└─────────────────────────────────────────────────────────┘
```

---

## 2. 核心类继承体系

```
BaseAgent (抽象基类)
├── 属性: name, systemPrompt, memory, llm, availableTools, printer, context
├── run(query)          → 主循环 (step × maxSteps)
├── step()              → 抽象方法
├── executeTool()       → 单工具执行
├── executeTools()      → 并发多工具执行 (CountDownLatch)
├── updateMemory()      → 消息管理
│
├── ReActAgent (抽象, Think-Act 模式)
│   ├── think()         → 抽象方法
│   ├── act()           → 抽象方法
│   ├── step()          → think() → act()
│   └── generateDigitalEmployee()  → 数字员工生成
│
│   ├── PlanningAgent (规划代理)
│   │   ├── think()     → 调用 LLM + PlanningTool
│   │   ├── act()       → 执行 planning 工具, 获取下一任务
│   │   ├── run()       → 注入 planPrePrompt, 调用 super.run()
│   │   └── getNextTask() → 解析 Plan, 返回当前步骤
│
│   └── ExecutorAgent (执行代理)
│       ├── think()     → 调用 LLM + 业务工具集
│       ├── act()       → 执行工具调用, 汇总结果
│       └── 支持: 单任务串行 / 多任务并行执行
│
└── SummaryAgent (总结代理)
    └── summaryTaskResult() → 总结执行结果, 提取文件列表
```

---

## 3. Plan-Solve 编排主流程

编排逻辑在 `PlanSolveHandlerImpl.handle()` 中实现：

```
                     用户 Query
                        │
                        ▼
                ┌──────────────┐
                │  SOP 召回     │  ← 可选: 匹配标准操作流程
                └──────┬───────┘
                       ▼
              ┌────────────────────┐
              │  创建 Agent 实例    │
              │  planning / executor│
              │  summary           │
              └────────┬───────────┘
                       ▼
         ╔═══════════════════════════════╗  ←── 迭代主循环
         ║  Step 1: PlanningAgent.run()  ║
         ║  生成/更新 Plan, 返回当前 task ║
         ╚═══════════╤═══════════════════╝
                     │
                     ▼
         ┌───────────────────────────┐
         │  task 包含 <sep> ?        │  ← 是否可并行拆分
         └─────┬───────────┬─────────┘
               │ YES       │ NO
               ▼           ▼
    ┌──────────────┐  ┌──────────────┐
    │ 并行执行      │  │ 单任务串行    │
    │ N 个 slave   │  │ executor     │
    │ ExecutorAgent│  │ .run(task)   │
    └──────┬───────┘  └──────┬───────┘
           │                 │
           ▼                 ▼
    ┌──────────────────────────────┐
    │  PlanningAgent.run(result)   │  ← 反馈执行结果, 获取下一 task
    │  更新 Plan 状态              │
    └──────────────┬───────────────┘
                   │
                   ▼
           ┌──────────────┐
           │  finish ?     │  ← Plan 所有步骤完成?
           └──┬───────┬───┘
              │ NO    │ YES
              │       ▼
              │  ┌──────────────┐
              │  │ SummaryAgent │
              │  │ .summary()   │  ← 总结任务结果
              │  └──────┬───────┘
              │         ▼
              │    SSE 输出 result
              │         │
              ▼         ▼
         继续迭代    流程结束
```

### 3.1 核心编排代码（伪代码）

```java
// PlanSolveHandlerImpl.handle()
PlanningAgent planning = new PlanningAgent(context);
ExecutorAgent executor = new ExecutorAgent(context);
SummaryAgent summary  = new SummaryAgent(context);

String planningResult = planning.run(query);  // 首次规划

while (stepIdx <= maxStepNum) {
    List<String> tasks = split(planningResult, "<sep>");
    
    String executorResult;
    if (tasks.size() == 1) {
        // 单任务: 直接执行
        executorResult = executor.run(tasks.get(0));
    } else {
        // 多任务: 并行执行 (CountDownLatch)
        // 每个 task 创建独立的 slaveExecutor
        // 执行完成后合并 memory
        executorResult = parallelExecute(tasks, executor);
    }
    
    // 反馈结果给 Planner, 获取下一步
    planningResult = planning.run(executorResult);
    
    if ("finish".equals(planningResult)) {
        summary.summaryTaskResult(executor.memory, query);
        break;
    }
    stepIdx++;
}
```

---

## 4. PlanningAgent 详细设计

### 4.1 职责

- **首次调用**: 接收用户 Query, 通过 LLM 调用 `PlanningTool` 创建 Plan（任务列表）
- **后续调用**: 接收 Executor 执行结果, 评估完成情况, 推进 Plan 到下一步
- **输出**: 当前需要执行的 task 描述（多个 task 用 `<sep>` 分隔表示可并行）

### 4.2 工具集

PlanningAgent **只拥有一个工具**: `PlanningTool`

```java
availableTools.addTool(planningTool);  // 仅 planning 工具
```

### 4.3 PlanningTool 命令体系

| 命令 | 参数 | 说明 |
|------|------|------|
| `create` | title, steps[] | 创建新 Plan |
| `update` | title?, steps[] | 更新已有 Plan |
| `mark_step` | step_index, step_status | 标记步骤状态 |
| `finish` | - | 标记 Plan 完成 |

### 4.4 Plan 数据模型

```java
public class Plan {
    String title;              // 计划标题
    List<String> steps;        // 步骤描述列表
    List<String> stepStatus;   // 状态: not_started / in_progress / completed / blocked
    List<String> notes;        // 步骤备注
}
```

**状态流转**:
```
not_started → in_progress → completed
                              ↑
                          (也可以是 blocked)
```

### 4.5 两种运行模式

通过配置 `planningCloseUpdate` 控制：

| 模式 | 值 | 行为 |
|------|-----|------|
| **动态更新** | `0` | 每次循环 LLM 重新评估 Plan, 可动态调整步骤 |
| **顺序执行** | `1` (默认) | Plan 创建后不再调用 LLM, 直接按序推进 `stepPlan()` |

**顺序执行模式流程**:
```
create → stepPlan(step0=in_progress) → 返回 step0
→ executor 执行 → stepPlan(step0=completed, step1=in_progress) → 返回 step1
→ ... → 所有 completed → finish
```

---

## 5. ExecutorAgent 详细设计

### 5.1 职责

- 接收 PlanningAgent 分配的 task
- 使用 ReAct 模式（Think-Act 循环）调用业务工具完成任务
- 返回执行结果给 PlanningAgent

### 5.2 工具集

ExecutorAgent 拥有**所有业务工具**:

```java
availableTools = context.getToolCollection();  // 全量工具
```

| 工具 | 功能 | 后端服务 |
|------|------|----------|
| `file_tool` | 文件读写管理 | 127.0.0.1:1601 |
| `deep_search` | 深度搜索 | 127.0.0.1:1601 |
| `report_tool` | HTML 报告生成 | 127.0.0.1:1601 |
| `code_interpreter` | 代码解释器 | 127.0.0.1:1601 |
| `data_analysis` | 数据分析 | 127.0.0.1:1601 |
| `mcp_tool` | MCP 协议工具 | 外部服务 |

### 5.3 Think-Act 流程

```
┌─────────────────────────────────────┐
│           think()                   │
│  1. 构建系统提示 (注入文件信息)       │
│  2. 调用 LLM.askTool()              │
│  3. 解析 ToolCall 响应              │
│  4. 无工具调用 → taskSummary        │
│  5. 有工具调用 → 记录到 memory      │
└──────────────┬──────────────────────┘
               ▼
┌─────────────────────────────────────┐
│           act()                     │
│  1. toolCalls 为空? → 直接返回      │
│  2. 并发执行所有 toolCalls          │
│  3. 生成数字员工 (可选)              │
│  4. 更新 memory                     │
│  5. 返回执行结果                    │
└─────────────────────────────────────┘
```

### 5.4 并行任务执行

当 Plan 返回多个用 `<sep>` 分隔的任务时，编排层会：

```java
// 为每个 task 创建独立的 slaveExecutor
for (String task : planningResults) {
    ExecutorAgent slaveExecutor = new ExecutorAgent(context);
    slaveExecutor.getMemory().addMessages(executor.getMemory().getMessages());
    // 并行执行
    ThreadUtil.execute(() -> {
        String taskResult = slaveExecutor.run(task);
        tmpTaskResult.put(task, taskResult);
        taskCount.countDown();
    });
}
// 合并 memory 回主 executor
```

---

## 6. LLM 交互层

### 6.1 双模式工具调用

LLM 类支持两种 Function Calling 模式：

| 模式 | functionCallType | 说明 |
|------|-----------------|------|
| **function_call** | 标准 | OpenAI 原生 Function Calling |
| **struct_parse** | 结构化解析 | LLM 输出 ```json``` 代码块, 正则提取 |

### 6.2 双供应商支持

- **OpenAI 兼容接口**: `callOpenAIFunctionCallStream()` → `data: [delta]` 格式
- **Claude 接口**: `callClaudeFunctionCallStream()` → `content_block_delta` 格式

### 6.3 流式输出

```
LLM Stream → SSEPrinter → SseEmitter → 前端
                │
                ├── message_type:
                │   ├── plan_thought   (规划思考)
                │   ├── plan           (Plan 结构)
                │   ├── task           (当前任务)
                │   ├── tool_thought   (工具调用思考)
                │   ├── task_summary   (任务总结)
                │   └── result         (最终结果)
                │
                └── interval 控制: firstInterval, sendInterval
```

---

## 7. SSE 事件处理管线

### 7.1 后端 → 前端事件流

```
PlanSolveHandlerImpl          SSEPrinter           Frontend
     │                            │                    │
     │  printer.send("plan_thought")                  │
     │───────────────────────────>│─── SSE ──────────>│ 规划思考中...
     │                            │                    │
     │  printer.send("plan", plan)                    │
     │───────────────────────────>│─── SSE ──────────>│ 显示任务列表
     │                            │                    │
     │  printer.send("task", step)                    │
     │───────────────────────────>│─── SSE ──────────>│ 高亮当前任务
     │                            │                    │
     │  printer.send("tool_thought")                  │
     │───────────────────────────>│─── SSE ──────────>│ 工具调用中...
     │                            │                    │
     │  printer.send("result", ...)                   │
     │───────────────────────────>│─── SSE ──────────>│ 最终结果
```

### 7.2 响应处理链

```
AgentResponse (从 Python 端 SSE)
     │
     ▼
PlanSolveAgentResponseHandler.handle()
     │
     ▼
BaseAgentResponseHandler.buildIncrResult()
     │
     ├── messageType == "plan_thought" → 事件: 规划思考
     ├── messageType == "plan"         → 事件: 计划生成/更新
     ├── messageType == "task"         → 事件: 新任务开始
     └── default                       → 事件: 子任务执行中
```

---

## 8. Memory 管理

### 8.1 消息类型

```java
public class Message {
    RoleType role;           // USER / SYSTEM / ASSISTANT / TOOL
    String content;          // 消息内容
    String base64Image;      // 图片 (多模态)
    String toolCallId;       // 工具调用 ID
    List<ToolCall> toolCalls; // 工具调用列表
}
```

### 8.2 多 Agent Memory 隔离与合并

```
PlanningAgent.memory ← 独立, 仅包含 planning 交互
ExecutorAgent.memory ← 主执行记忆
    ├── 并行时: 每个 slaveExecutor 独立 memory
    └── 合并时: 从 memoryIndex 开始追加回主 executor
```

### 8.3 上下文截断

Executor 设置 `maxObserve` (默认 10000) 截断工具返回结果，避免 token 溢出。

---

## 9. 配置体系

所有配置通过 `GenieConfig` + `application.yml` 管理：

```yaml
autobots:
  autoagent:
    planner:
      model_name: "glm-5.1"          # Planner 使用的模型
      max_steps: 40                    # 最大规划步数
      system_prompt: {...}             # 可覆盖的 System Prompt
      next_step_prompt: {...}          # 可覆盖的 Next Step Prompt
      close_update: "1"                # 1=顺序执行, 0=动态更新
      pre_prompt: "分析问题并制定计划："  # 首次规划前缀
    executor:
      model_name: "glm-5.1"          # Executor 使用的模型
      max_steps: 40
      max_observe: 10000              # 工具结果截断长度
    summary:
      system_prompt: "..."
      message_size_limit: 1000
    tool:
      plan_tool:
        desc: "..."                   # PlanningTool 描述
        params: {...}                 # 可覆盖的工具参数
```

**关键设计**: Planner 和 Executor 可使用不同的模型，允许用强模型做规划、轻量模型做执行。

---

## 10. SOP（标准操作流程）集成

```
用户 Query
    │
    ▼
SopRecallService.sopRecall(query)
    │
    ▼
匹配到 SOP?
    ├── YES → 注入 sopPrompt 到 AgentContext
    │         → Planner/Executor System Prompt 中的 {{sopPrompt}} 被替换
    └── NO  → 继续, 不影响主流程 (异常不阻塞)
```

SOP 提供了一种「注入领域知识」的机制，让 LLM 遵循预设的最佳实践流程。

---

## 11. 完整调用链路（时序图）

```
Frontend          GenieController     PlanSolveHandler    PlanningAgent    ExecutorAgent    SummaryAgent
   │                   │                    │                  │                │               │
   │── POST /AutoAgent─>│                    │                  │                │               │
   │                   │── handle() ────────>│                  │                │               │
   │                   │                    │── new Planning ─>│                │               │
   │                   │                    │── new Executor ──┼───────────────>│               │
   │                   │                    │── new Summary ───┼────────────────┼──────────────>│
   │                   │                    │                  │                │               │
   │<── SSE heartbeat ─│                    │                  │                │               │
   │                   │                    │── planning.run()─>│               │               │
   │                   │                    │                  │── think() ────>│               │
   │<── SSE plan_thought                    │                  │  (LLM+PlanningTool)             │
   │<── SSE plan ──────│                    │                  │── act()        │               │
   │                   │                    │<── "task1<sep>task2"               │               │
   │                   │                    │                  │                │               │
   │<── SSE task ──────│                    │── executor.run()─┼───────────────>│               │
   │                   │                    │                  │                │── think()     │
   │<── SSE tool_thought                    │                  │                │── act()       │
   │                   │                    │                  │                │── think()     │
   │<── SSE task_summary                    │                  │                │── act()       │
   │                   │                    │<── executorResult────────────────│               │
   │                   │                    │                  │                │               │
   │                   │                    │── planning.run()─>│               │               │
   │<── SSE plan ──────│                    │                  │ (评估结果)     │               │
   │                   │                    │<── "finish" ─────│               │               │
   │                   │                    │                  │                │               │
   │                   │                    │── summary.summary()───────────────────────────────>│
   │<── SSE result ────│                    │<── TaskSummaryResult─────────────────────────────│
   │                   │                    │                  │                │               │
```

---

## 12. 架构亮点与设计模式总结

| 设计点 | 实现方式 | 优势 |
|--------|---------|------|
| **Plan-Solve 分离** | PlanningAgent 只做规划, ExecutorAgent 只做执行 | 职责单一, 易于调试 |
| **ReAct 模式** | Think → Act 循环 | 可解释的推理过程 |
| **并行执行** | `<sep>` 分隔 + CountDownLatch | 多任务并发, 提升效率 |
| **工具隔离** | Planner 只有 PlanningTool, Executor 有全量工具 | 防止 Planner 陷入执行细节 |
| **Memory 合并** | slaveExecutor → 主 executor | 并行任务记忆统一管理 |
| **SOP 注入** | 外置标准流程模板 | 领域知识可配置化 |
| **双模型支持** | Planner/Executor 可配不同模型 | 成本与质量的平衡 |
| **策略模式路由** | AgentHandlerFactory + support() | 新增 Agent 类型零改动 |
| **流式输出** | SSEPrinter + message_type 分类 | 前端实时渲染规划/执行过程 |
| **配置外置** | GenieConfig + YAML 全覆盖 | Prompt/参数热更新 |

---

## 13. 文件索引

| 文件路径 | 说明 |
|---------|------|
| `agent/agent/BaseAgent.java` | Agent 抽象基类, 定义 run/step/memory |
| `agent/agent/ReActAgent.java` | ReAct 模式抽象层, 定义 think/act |
| `agent/agent/PlanningAgent.java` | 规划代理, 管理 Plan 生命周期 |
| `agent/agent/ExecutorAgent.java` | 执行代理, ReAct 循环调用业务工具 |
| `agent/agent/SummaryAgent.java` | 总结代理, 生成最终摘要 |
| `agent/agent/AgentContext.java` | Agent 共享上下文 |
| `agent/dto/Plan.java` | Plan 数据模型, 状态管理 |
| `agent/dto/Message.java` | 消息模型 (多角色/多模态) |
| `agent/tool/common/PlanningTool.java` | 计划工具 (create/update/mark_step/finish) |
| `agent/prompt/PlanningPrompt.java` | Planner 默认 Prompt 模板 |
| `agent/prompt/ToolCallPrompt.java` | Executor 默认 Prompt 模板 |
| `agent/llm/LLM.java` | LLM 调用层 (OpenAI/Claude 双协议) |
| `agent/printer/SSEPrinter.java` | SSE 流式输出 |
| `service/impl/PlanSolveHandlerImpl.java` | **核心编排器** - Plan-Solve 主循环 |
| `handler/PlanSolveAgentResponseHandler.java` | SSE 响应处理器 |
| `config/GenieConfig.java` | 全局配置中心 |

---

☕️ 制作不易，请我喝咖啡☕️关注我➕