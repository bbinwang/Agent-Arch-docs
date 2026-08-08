# 05-03 — fs_explorer/workflow.py 代码讲解

> **文件**: `src/fs_explorer/workflow.py`
> **行数**: ~215 行
> **职责**: 工作流编排，定义 4 步状态机和事件类型
> **预估字数**: ~10,000 字

---

## 1. 文件概览

`workflow.py` 实现了基于 `llama-index-workflows` 框架的 4 步状态机，是 Agent 决策循环的核心编排器。文件定义了工作流状态模型、6 种事件类型、以及 4 个步骤函数。

---

## 2. 导入声明

```python
from workflows import Workflow, Context, step
from workflows.events import StartEvent, StopEvent, Event, InputRequiredEvent, HumanResponseEvent
from workflows.resource import Resource
from pydantic import BaseModel
from typing import Annotated, cast, Any

from .agent import FsExplorerAgent
from .models import GoDeeperAction, ToolCallAction, StopAction, AskHumanAction
from .fs import describe_dir_content
```

**导入分析**:
- `workflows`: llama-index-workflows 框架核心
- `pydantic`: 状态模型定义
- `typing`: 类型注解
- 内部导入: Agent 实例、Action 类型、目录描述函数

---

## 3. 全局实例

```python
AGENT = FsExplorerAgent()
```

**设计**: 全局单例 Agent 实例，通过 `Resource(get_agent)` 注入到工作流步骤。

**⚠️ 问题**: 全局可变状态，不利于测试和并发。建议通过参数传入。

---

## 4. WorkflowState 状态模型

```python
class WorkflowState(BaseModel):
    intial_task: str = ""
    current_directory: str = "."
```

**字段**:
- `intial_task`: 用户原始任务（注意拼写错误，应为 `initial_task`）
- `current_directory`: 当前探索目录，默认为 `"."`

**设计**: 使用 Pydantic BaseModel 确保类型安全，通过 `Context.store` 管理。

---

## 5. 事件类型定义

### 5.1 事件继承体系

```python
class InputEvent(StartEvent):
    """工作流启动事件"""
    task: str

class GoDeeperEvent(Event):
    """进入子目录事件"""
    directory: str
    reason: str

class ToolCallEvent(Event):
    """工具调用事件"""
    tool_name: str
    tool_input: dict[str, Any]
    reason: str

class AskHumanEvent(InputRequiredEvent):
    """请求用户输入事件"""
    question: str
    reason: str

class HumanAnswerEvent(HumanResponseEvent):
    """用户回答事件"""
    response: str

class ExplorationEndEvent(StopEvent):
    """工作流结束事件"""
    final_result: str | None = None
    error: str | None = None
```

### 5.2 事件分类

| 事件 | 基类 | 用途 | 流向 |
|------|------|------|------|
| InputEvent | StartEvent | 启动工作流 | 外部 → start_exploration |
| GoDeeperEvent | Event | 目录深入 | start/go_deeper/tool_call/human → go_deeper_action |
| ToolCallEvent | Event | 工具调用 | start/go_deeper/tool_call/human → tool_call_action |
| AskHumanEvent | InputRequiredEvent | 请求用户输入 | start/go_deeper/tool_call/human → [等待] |
| HumanAnswerEvent | HumanResponseEvent | 用户回答 | [用户] → receive_human_answer |
| ExplorationEndEvent | StopEvent | 结束工作流 | 任何步骤 → 结束 |

### 5.3 设计要点

1. **继承框架事件**: `StartEvent`、`StopEvent`、`InputRequiredEvent`、`HumanResponseEvent` 是框架预定义的事件基类，提供生命周期管理。

2. **携带推理原因**: `GoDeeperEvent`、`ToolCallEvent`、`AskHumanEvent` 都包含 `reason` 字段，用于向用户解释 Agent 的决策动机。

3. **输入/输出事件对**: `AskHumanEvent`（InputRequiredEvent）和 `HumanAnswerEvent`（HumanResponseEvent）形成一对请求-响应事件，支持 Human-in-the-Loop。

---

## 6. Resource 依赖注入

```python
def get_agent(*args, **kwargs) -> FsExplorerAgent:
    return AGENT
```

**设计**: 工厂函数返回全局 Agent 实例，通过 `Annotated[FsExplorerAgent, Resource(get_agent)]` 注入到步骤。

**作用**: 
- 步骤函数声明依赖，由工作流引擎负责注入
- 便于测试时替换为 Mock 实例

---

## 7. FsExplorerWorkflow 类

### 7.1 类定义

```python
class FsExplorerWorkflow(Workflow):
    ...
```

**继承**: `Workflow` 基类，提供 `run()`、`stream_events()` 等核心方法。

### 7.2 步骤 1: start_exploration

```python
@step
async def start_exploration(
    self,
    ev: InputEvent,
    ctx: Context[WorkflowState],
    agent: Annotated[FsExplorerAgent, Resource(get_agent)],
) -> ExplorationEndEvent | GoDeeperEvent | ToolCallEvent | AskHumanEvent:
    async with ctx.store.edit_state() as state:
        state.intial_task = ev.task
    dirdescription = describe_dir_content(".")
    agent.configure_task(
        f"Given that the current directory ('.') looks like this:\n\n```text\n{dirdescription}```\n\n"
        f"And that the user is giving you this task: '{ev.task}', what action should you take first?"
    )
    result = await agent.take_action()
    if result is None:
        return ExplorationEndEvent(error="Could not produce action to take")
    action, action_type = result
    if action_type == "godeeper":
        godeeper = cast(GoDeeperEvent, action.action)
        res = GoDeeperEvent(directory=godeeper.directory, reason=action.reason)
        async with ctx.store.edit_state() as state:
            state.current_directory = godeeper.directory
        ctx.write_event_to_stream(res)
    elif action_type == "toolcall":
        toolcall = cast(ToolCallAction, action.action)
        res = ToolCallEvent(tool_name=toolcall.tool_name, tool_input=toolcall.to_fn_args(), reason=action.reason)
        ctx.write_event_to_stream(res)
    elif action_type == "askhuman":
        askhuman = cast(AskHumanAction, action.action)
        res = AskHumanEvent(question=askhuman.question, reason=action.reason)
    else:
        stopaction = cast(StopAction, action.action)
        res = ExplorationEndEvent(final_result=stopaction.final_result)
    return res
```

**功能**: 初始探索步骤，描述当前目录，获取首个 Action。

**执行流程**:
1. 保存初始任务到状态
2. 描述当前目录（`describe_dir_content(".")`）
3. 配置 Agent 任务（包含目录描述和用户任务）
4. 获取 Agent 决策
5. 根据 Action 类型路由到对应事件

**状态更新**:
- `state.intial_task = ev.task` — 保存初始任务
- `state.current_directory = godeeper.directory` — 仅在 godeeper 时更新

**事件路由**:

| Action 类型 | 产生的事件 | 状态更新 | 写入流 |
|------------|-----------|---------|--------|
| godeeper | GoDeeperEvent | current_directory | ✅ |
| toolcall | ToolCallEvent | - | ✅ |
| askhuman | AskHumanEvent | - | ❌（默认写入） |
| stop | ExplorationEndEvent | - | ❌（StopEvent 默认） |

**注意**: `AskHumanEvent` 和 `ExplorationEndEvent` 不需要显式调用 `write_event_to_stream()`，因为它们分别继承自 `InputRequiredEvent` 和 `StopEvent`，框架会自动写入流。

### 7.3 步骤 2: go_deeper_action

```python
@step
async def go_deeper_action(
    self,
    ev: GoDeeperEvent,
    ctx: Context[WorkflowState],
    agent: Annotated[FsExplorerAgent, Resource(get_agent)],
) -> ExplorationEndEvent | ToolCallEvent | GoDeeperEvent | AskHumanEvent:
    state = await ctx.store.get_state()
    dirdescription = describe_dir_content(state.current_directory)
    agent.configure_task(
        f"Given that the current directory ('{state.current_directory}') looks like this:\n\n"
        f"```text\n{dirdescription}```\n\n"
        f"And that the user is giving you this task: '{state.intial_task}', what action should you take next?"
    )
    result = await agent.take_action()
    # ... 路由逻辑同 start_exploration
```

**功能**: 进入子目录后重新描述并决策。

**与 start_exploration 的区别**:
- 使用 `state.current_directory` 而非 `"."`
- 提示词使用 "what action should you take next?" 而非 "first"
- 不更新 `intial_task`（已在 start 中设置）

### 7.4 步骤 3: tool_call_action

```python
@step
async def tool_call_action(
    self,
    ev: ToolCallEvent,
    ctx: Context[WorkflowState],
    agent: Annotated[FsExplorerAgent, Resource(get_agent)],
) -> ExplorationEndEvent | ToolCallEvent | GoDeeperEvent | AskHumanEvent:
    agent.configure_task("Given the result from the tool call you just performed, what action should you take next?")
    result = await agent.take_action()
    # ... 路由逻辑同上
```

**功能**: 工具调用后基于结果继续决策。

**与前两步的区别**:
- 不重新描述目录
- 提示词仅告知 "基于工具结果决定下一步"

### 7.5 步骤 4: receive_human_answer

```python
@step
async def receive_human_answer(
    self,
    ev: HumanAnswerEvent,
    ctx: Context[WorkflowState],
    agent: Annotated[FsExplorerAgent, Resource(get_agent)],
) -> ExplorationEndEvent | ToolCallEvent | GoDeeperEvent | AskHumanEvent:
    state = await ctx.store.get_state()
    agent.configure_task(
        f"Human response to your question: {ev.response}\n\n"
        f"Based on it, proceed with you exploration based on the original task: {state.intial_task}"
    )
    result = await agent.take_action()
    # ... 路由逻辑同上
```

**功能**: 接收用户回答后继续决策。

**提示词设计**:
- 包含用户回答（`ev.response`）
- 提醒原始任务（`state.intial_task`）

---

## 8. 代码重复问题分析

⚠️ **严重代码重复**: 四个步骤中的事件路由逻辑完全相同：

```python
# 这段代码在四个步骤中重复出现
if action_type == "godeeper":
    godeeper = cast(GoDeeperAction, action.action)
    res = GoDeeperEvent(directory=godeeper.directory, reason=action.reason)
    async with ctx.store.edit_state() as state:
        state.current_directory = godeeper.directory
    ctx.write_event_to_stream(res)
elif action_type == "toolcall":
    toolcall = cast(ToolCallAction, action.action)
    res = ToolCallEvent(tool_name=toolcall.tool_name, tool_input=toolcall.to_fn_args(), reason=action.reason)
    ctx.write_event_to_stream(res)
elif action_type == "askhuman":
    askhuman = cast(AskHumanAction, action.action)
    res = AskHumanEvent(question=askhuman.question, reason=action.reason)
else:
    stopaction = cast(StopAction, action.action)
    res = ExplorationEndEvent(final_result=stopaction.final_result)
```

**改进建议**: 提取为共享方法

```python
def _route_action(self, action: Action, ctx: Context[WorkflowState]):
    action_type = action.to_action_type()
    if action_type == "godeeper":
        godeeper = cast(GoDeeperAction, action.action)
        res = GoDeeperEvent(directory=godeeper.directory, reason=action.reason)
        async with ctx.store.edit_state() as state:
            state.current_directory = godeeper.directory
        ctx.write_event_to_stream(res)
    # ... 其他分支
    return res
```

---

## 9. 工作流实例化

```python
workflow = FsExplorerWorkflow(timeout=120)
```

**配置**:
- `timeout=120`: 工作流超时时间 120 秒

**使用**: 在 `main.py` 中通过 `workflow.run(start_event=InputEvent(task=task))` 启动。

---

## 10. 设计模式分析

### 10.1 State Machine 模式

四个步骤对应四个状态，事件驱动状态转换：

```
[start_exploration] ──GoDeeperEvent──→ [go_deeper_action]
[start_exploration] ──ToolCallEvent──→ [tool_call_action]
[start_exploration] ──AskHumanEvent──→ [等待用户输入]
[start_exploration] ──ExplorationEndEvent──→ [结束]
```

### 10.2 Template Method 模式

四个步骤共享相同的决策后处理逻辑（获取 Action → 路由事件），只是前置的上下文准备不同。

### 10.3 Observer 模式

`ctx.write_event_to_stream()` 将事件写入流，外部观察者（CLI）通过 `stream_events()` 订阅。

---

## 11. 复杂度分析

| 操作 | 时间复杂度 | 空间复杂度 |
|------|-----------|-----------|
| 状态更新 | O(1) | O(1) |
| 目录描述 | O(n) | O(n) | n = 目录中文件数 |
| Agent 决策 | O(m) + API | O(m) | m = 对话历史长度 |
| 事件路由 | O(1) | O(1) |

---

## 12. 改进建议

### 12.1 消除代码重复

如上节所述，提取 `_route_action()` 共享方法。

### 12.2 状态模型修正

```python
class WorkflowState(BaseModel):
    intial_task: str = ""  # 拼写错误，应为 initial_task
```

### 12.3 全局状态

```python
AGENT = FsExplorerAgent()  # 全局可变状态
```

建议通过构造函数传入，便于测试和并发。

### 12.4 超时后行为

当前超时后工作流直接终止，没有友好的错误提示。建议捕获超时异常并返回带有错误信息的 `ExplorationEndEvent`。

---

# ❤️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../../res/promotion.jpg)