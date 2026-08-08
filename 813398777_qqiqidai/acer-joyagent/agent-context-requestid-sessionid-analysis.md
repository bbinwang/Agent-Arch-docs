# AgentContext 中 requestId vs sessionId 分析

## 1. 定义与赋值

### 源码位置：`GenieController.java` 第 124-126 行

```java
AgentContext agentContext = AgentContext.builder()
        .requestId(request.getRequestId())
        .sessionId(request.getRequestId())  // ⚠️ 初始值与 requestId 相同
        // ...
        .build();
```

**结论：在 `/AutoAgent` 入口处，`requestId` 和 `sessionId` 被赋为同一个值。**

两者均来源于 `AgentRequest.requestId`，由调用方（前端或网关）在每次 HTTP 请求中生成并传入。

---

## 2. 语义差异

| 维度 | `requestId` | `sessionId` |
|------|-------------|-------------|
| **语义** | **单次请求的唯一标识**，用于日志追踪、链路关联 | **会话的持久标识**，用于多轮对话中的上下文/文件隔离 |
| **生命周期** | 一次 API 调用 | 一次完整对话（可能跨越多轮） |
| **生成方** | 前端/网关，每次请求生成 | 理论上应在会话创建时生成，多轮共享 |
| **当前赋值** | `request.getRequestId()` | `request.getRequestId()`（⚠️ 与 requestId 相同） |

---

## 3. 实际使用场景

### 3.1 `requestId` 的用途 — 日志追踪 + SSE 关联

```java
// SSE 心跳和连接监控
startHeartbeat(emitter, request.getRequestId());
registerSSEMonitor(emitter, request.getRequestId(), heartbeatFuture);

// 工具执行日志
log.info("requestId:{} task:{} toolName:{}", agentContext.getRequestId(), ...);

// SSE 输出事件
.requestId(request.getRequestId())  // SSEPrinter 中
```

**特征**：`requestId` 在整个 backend 内部仅用于**可观测性**（日志、SSE 事件标记），不参与业务逻辑。

### 3.2 `sessionId` 的用途 — 跨工具的业务数据隔离（多轮对话）

这是两者的**核心差异**所在：

```java
// CodeInterpreterTool.java:71 — 传给 genie-tool 的 requestId 实际用的是 sessionId
.requestId(agentContext.getSessionId()) // 适配多轮对话

// FileTool.java:111 — 文件上传同理
// 构建请求体 多轮对话替换requestId为sessionId
fileRequest.setRequestId(agentContext.getSessionId());

// ReportTool.java:85 — 报告生成同理
.requestId(agentContext.getSessionId()) // 适配多轮对话
```

**注释已经明确说明了意图**：`sessionId` 用于"适配多轮对话"。

### 3.3 genie-tool 端 `requestId` 的业务含义

在 genie-tool Python 微服务中，收到的 `requestId` 实际上是 backend 传来的 `sessionId`，它被用于：

| 用途 | 代码位置 | 说明 |
|------|----------|------|
| **文件存储 scope** | `file_table_op.py:49` | `FileDB.save(filename, content, scope=request_id)` — 按 sessionId 隔离文件命名空间 |
| **文件查询** | `file_table_op.py:110` | `get_by_request_id(request_id)` — 查找该会话的所有上传文件 |
| **文件 ID 生成** | `protocal.py:53-54` | `md5(requestId + fileName)` — 文件唯一标识由 sessionId + 文件名决定 |
| **代码沙箱文件路径** | `code_interpreter.py:97/107` | `request_id` 作为代码执行的工作目录隔离依据 |

---

## 4. 架构示意图

```
前端/网关
  │
  │  AgentRequest { requestId: "uuid-1" }
  ▼
GenieController (/AutoAgent)
  │
  ├─ requestId = "uuid-1"     ← 日志/SSE/心跳（可观测性）
  ├─ sessionId = "uuid-1"     ← 业务隔离（文件/代码沙箱）
  │
  ▼
ReActAgent 循环
  │
  ├─ FileTool.execute()
  │    └─ fileRequest.setRequestId(agentContext.getSessionId())  ← "uuid-1"
  │         │
  │         ▼  OkHttp → genie-tool:1601
  │    genie-tool 以 "uuid-1" 为 scope 存储文件
  │
  ├─ CodeInterpreterTool.execute()
  │    └─ request.setRequestId(agentContext.getSessionId())      ← "uuid-1"
  │         │
  │         ▼  OkHttp → genie-tool:1601
  │    genie-tool 以 "uuid-1" 为工作目录执行代码
  │
  └─ ReportTool.execute()
       └─ request.setRequestId(agentContext.getSessionId())      ← "uuid-1"
```

---

## 5. 当前问题：两者同值

### 现状

`GenieController` 中 `requestId` 和 `sessionId` 被赋了同一个值：

```java
.requestId(request.getRequestId())
.sessionId(request.getRequestId())
```

这意味着**当前代码实际上没有真正的多轮对话隔离**。每一轮请求都带着前端生成的新 `requestId`，这个值同时充当了 `sessionId`。

### 代码中已有的"适配"痕迹

工具层代码中有多处注释表明开发者**意识到了这个问题**并做了适配：

```java
// FileTool.java:110
// 构建请求体 多轮对话替换requestId为sessionId
fileRequest.setRequestId(agentContext.getSessionId());

// CodeInterpreterTool.java:71
.requestId(agentContext.getSessionId()) // 适配多轮对话
```

这说明设计意图是：
- 传给下游工具的应该是 `sessionId`（跨轮次稳定）
- 但由于当前两者同值，这个适配还没真正生效

### 真正多轮对话时应该怎么做

如果 `AgentRequest` 能携带独立的 `sessionId`（由前端在会话创建时生成，多轮共享），则：

```
第1轮：AgentRequest { requestId: "req-001", sessionId: "session-abc" }
第2轮：AgentRequest { requestId: "req-002", sessionId: "session-abc" }
第3轮：AgentRequest { requestId: "req-003", sessionId: "session-abc" }
```

- `requestId` 每轮不同 → 日志中可区分每轮调用
- `sessionId` 全程相同 → genie-tool 中文件/代码沙箱在同一命名空间内共享

---

## 6. 改进建议

### 短期：AgentRequest 增加 sessionId 字段

```java
public class AgentRequest {
    private String requestId;   // 每次请求唯一
    private String sessionId;   // 会话级别唯一，多轮共享
    // ...
}
```

Controller 中分别赋值：

```java
AgentContext agentContext = AgentContext.builder()
        .requestId(request.getRequestId())
        .sessionId(request.getSessionId() != null ? request.getSessionId() : request.getRequestId())
        .build();
```

### 中期：genie-tool 中重命名 requestId → sessionId

避免语义混淆，genie-tool 中接收的参数应明确为 `session_id`，因为它的实际用途是会话级隔离而非请求级追踪。

### 长期：genie-tool 侧 file_id 与 request_id 解耦

当前代码中已有 TODO 标注：

```python
# file_manage.py:75
# TODO 目前 file_id 实际上是 request_id，后续统一修改
```

建议将 `file_id` 独立为 UUID，`session_id` 仅作为查询过滤条件。

---

☕️ 制作不易，请我喝咖啡☕️关注我➕