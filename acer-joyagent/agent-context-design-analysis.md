# AgentContext 设计分析

## 一、requestId vs sessionId 语义差异

### 1.1 定义与赋值

**源码位置：** `GenieController.java` 第 124-126 行

```java
AgentContext agentContext = AgentContext.builder()
        .requestId(request.getRequestId())
        .sessionId(request.getRequestId())  // ⚠️ 初始值与 requestId 相同
        // ...
        .build();
```

**结论：在 `/AutoAgent` 入口处，`requestId` 和 `sessionId` 被赋为同一个值。**

两者均来源于 `AgentRequest.requestId`，由调用方（前端或网关）在每次 HTTP 请求中生成并传入。

### 1.2 语义差异

| 维度 | `requestId` | `sessionId` |
|------|-------------|-------------|
| **语义** | **单次请求的唯一标识**，用于日志追踪、链路关联 | **会话的持久标识**，用于多轮对话中的上下文/文件隔离 |
| **生命周期** | 一次 API 调用 | 一次完整对话（可能跨越多轮） |
| **生成方** | 前端/网关，每次请求生成 | 理论上应在会话创建时生成，多轮共享 |
| **当前赋值** | `request.getRequestId()` | `request.getRequestId()`（⚠️ 与 requestId 相同） |

### 1.3 实际使用场景

#### requestId — 日志追踪 + SSE 关联

```java
// SSE 心跳和连接监控
startHeartbeat(emitter, request.getRequestId());
registerSSEMonitor(emitter, request.getRequestId(), heartbeatFuture);

// 工具执行日志
log.info("requestId:{} task:{} toolName:{}", agentContext.getRequestId(), ...);

// SSE 输出事件（SSEPrinter 中）
.requestId(request.getRequestId())
```

**特征**：`requestId` 在整个 backend 内部仅用于**可观测性**（日志、SSE 事件标记），不参与业务逻辑。

#### sessionId — 跨工具的业务数据隔离（多轮对话）

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

### 1.4 genie-tool 端 requestId 的业务含义

在 genie-tool Python 微服务中，收到的 `requestId` 实际上是 backend 传来的 `sessionId`，它被用于：

| 用途 | 代码位置 | 说明 |
|------|----------|------|
| **文件存储 scope** | `file_table_op.py:49` | `FileDB.save(filename, content, scope=request_id)` — 按 sessionId 隔离文件命名空间 |
| **文件查询** | `file_table_op.py:110` | `get_by_request_id(request_id)` — 查找该会话的所有上传文件 |
| **文件 ID 生成** | `protocal.py:53-54` | `md5(requestId + fileName)` — 文件唯一标识由 sessionId + 文件名决定 |
| **代码沙箱文件路径** | `code_interpreter.py:97/107` | `request_id` 作为代码执行的工作目录隔离依据 |

### 1.5 当前问题：两者同值

`GenieController` 中 `requestId` 和 `sessionId` 被赋了同一个值：

```java
.requestId(request.getRequestId())
.sessionId(request.getRequestId())
```

这意味着**当前代码实际上没有真正的多轮对话隔离**。每一轮请求都带着前端生成的新 `requestId`，这个值同时充当了 `sessionId`。

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

### 1.6 改进建议

**短期**：AgentRequest 增加 `sessionId` 字段

```java
public class AgentRequest {
    private String requestId;   // 每次请求唯一
    private String sessionId;   // 会话级别唯一，多轮共享
}
```

**长期**：genie-tool 侧 `file_id` 与 `request_id` 解耦（代码中已有 TODO 标注：`file_manage.py:75`）

---

## 二、buildToolCollection 实现

### 2.1 源码

**位置：** `GenieController.java` 第 179-261 行

```java
private ToolCollection buildToolCollection(AgentContext agentContext, AgentRequest request) {
    ToolCollection toolCollection = new ToolCollection();
    toolCollection.setAgentContext(agentContext);

    // ① dataAgent 模式
    if ("dataAgent".equals(request.getOutputStyle())) {
        toolCollection.addTool(new ReportTool(agentContext));
        toolCollection.addTool(new DataAnalysisTool(agentContext));
    } else {
    // ② 通用模式
        toolCollection.addTool(new FileTool(agentContext));

        List<String> agentToolList = Arrays.asList(
            genieConfig.getMultiAgentToolListMap()
                .getOrDefault("default", "search,code,report").split(",")
        );
        // 按 name 匹配动态注册
        if (agentToolList.contains("code"))        toolCollection.addTool(new CodeInterpreterTool(agentContext));
        if (agentToolList.contains("report"))      toolCollection.addTool(new ReportTool(agentContext));
        if (agentToolList.contains("search"))      toolCollection.addTool(new DeepSearchTool(agentContext));
        if (agentToolList.contains("data_analysis")) toolCollection.addTool(new DataAnalysisTool(agentContext));
    }

    // ③ MCP 工具 — 动态发现
    try {
        McpTool mcpTool = new McpTool(agentContext);
        for (String mcpServer : genieConfig.getMcpServerUrlArr()) {
            String listToolResult = mcpTool.listTool(mcpServer);
            // 解析返回，逐个注册
            for (...) {
                toolCollection.addMcpTool(method, description, inputSchema, mcpServer);
            }
        }
    } catch (Exception e) { ... }

    return toolCollection;
}
```

### 2.2 执行流程图

```
buildToolCollection(agentContext, request)
│
├─ 创建 ToolCollection { toolMap: HashMap, mcpToolMap: HashMap }
│
├─ 判断 outputStyle
│   │
│   ├─ "dataAgent" ─────────────────────────────┐
│   │   ├─ ReportTool        → toolMap["report_tool"]        │
│   │   └─ DataAnalysisTool  → toolMap["data_analysis"]      │
│   │                                                     │ 两种模式互斥
│   └─ 其他（通用模式）─────────────────────────────────┘
│       ├─ FileTool           → toolMap["file_tool"]           (必装)
│       ├─ 从配置读取 "default" → "search,code,report"
│       │   ├─ "code"         → CodeInterpreterTool
│       │   ├─ "report"       → ReportTool
│       │   ├─ "search"       → DeepSearchTool
│       │   └─ "data_analysis"→ DataAnalysisTool
│
├─ MCP 动态发现（遍历 mcpServerUrlArr 配置项）
│   ├─ mcpTool.listTool(serverUrl) → 拉取远端工具列表
│   └─ 逐个 addMcpTool(name, desc, schema, url)
│       → mcpToolMap["tool_name"] = McpToolInfo{...}
│
└─ 返回 ToolCollection
```

### 2.3 两种工具的调度机制

`ToolCollection.execute(name, input)` 在 ReAct 循环中被调用：

```java
// ToolCollection.java:75-88
public Object execute(String name, Object toolInput) {
    if (toolMap.containsKey(name)) {
        // 内置工具：直接调用 BaseTool.execute()
        return getTool(name).execute(toolInput);
    } else if (mcpToolMap.containsKey(name)) {
        // MCP 工具：通过 HTTP 转发到 MCP Server
        McpToolInfo toolInfo = mcpToolMap.get(name);
        McpTool mcpTool = new McpTool();
        mcpTool.setAgentContext(agentContext);
        return mcpTool.callTool(toolInfo.getMcpServerUrl(), name, toolInput);
    }
    return null;
}
```

| 类型 | 存储结构 | 执行方式 | 通信协议 |
|------|----------|----------|----------|
| **内置工具** | `Map<String, BaseTool> toolMap` | 本地 `tool.execute(input)` | Java 方法调用 |
| **MCP 工具** | `Map<String, McpToolInfo> mcpToolMap` | HTTP 转发 `mcpTool.callTool(url, name, input)` | OkHttp → MCP Server |

### 2.4 工具注册表总览

| 工具名 | 类 | 模式 | 说明 |
|--------|-----|------|------|
| `file_tool` | `FileTool` | 通用（必装） | 文件上传/读取/下载 |
| `code_interpreter` | `CodeInterpreterTool` | 通用（可配） | 代码沙箱执行 |
| `report_tool` | `ReportTool` | 通用（可配）/ dataAgent（必装） | 报告/HTML 生成 |
| `deep_search` | `DeepSearchTool` | 通用（可配） | 联网深度搜索 |
| `data_analysis` | `DataAnalysisTool` | 通用（可配）/ dataAgent（必装） | 自动数据分析 |
| 动态 MCP 工具 | `McpToolInfo` | 两种模式均可 | 通过 MCP 协议动态注册 |

配置来源：`GenieConfig.multiAgentToolListMap`，默认值为 `"search,code,report"`。

---

## 三、@Accessors(chain = true) 优化建议

### 3.1 当前写法的问题

Controller 中创建 AgentContext 需要两步完成：

```java
// 第一步：builder 创建（14 个字段）
AgentContext agentContext = AgentContext.builder()
        .requestId(...)
        .sessionId(...)
        .query(...)
        .task("")
        .dateInfo(...)
        .productFiles(new ArrayList<>())
        .taskProductFiles(new ArrayList<>())
        .sopPrompt(...)
        .basePrompt(...)
        .agentType(...)
        .isStream(...)
        .templateType(...)
        .build();

// 第二步：拆出来单独 set（因为 buildToolCollection 依赖 agentContext 自身）
agentContext.setToolCollection(buildToolCollection(agentContext, request));
```

**断裂点**：Builder 构建时 `buildToolCollection(agentContext, request)` 引用了还没 build 好的 `this`，所以只能 build 完再补 set。

### 3.2 加 @Accessors(chain = true) 后

```java
AgentContext agentContext = new AgentContext()
        .setRequestId(request.getRequestId())
        .setSessionId(request.getRequestId())
        .setQuery(request.getQuery())
        .setTask("")
        .setDateInfo(DateUtil.CurrentDateInfo())
        .setProductFiles(new ArrayList<>())
        .setTaskProductFiles(new ArrayList<>())
        .setSopPrompt(request.getSopPrompt())
        .setBasePrompt(request.getBasePrompt())
        .setAgentType(request.getAgentType())
        .setIsStream(Objects.nonNull(request.getIsStream()) ? request.getIsStream() : false)
        .setTemplateType("dataAgent".equals(request.getOutputStyle()) ? "fix" : "empty")
        .setToolCollection(buildToolCollection(agentContext, request));  // ✅ 自引用也行
```

**一个连续链完成**，不存在"先 build 再补 set"的断裂。

### 3.3 优势对比

| 维度 | Builder（当前） | chain setter |
|------|----------------|--------------|
| **自引用场景** | ❌ builder 里引用不了还没 build 好的 `this` | ✅ 天然支持，`this` 已经存在 |
| **null 字段** | 必须显式写 `.task("")`、`.productFiles(new ArrayList<>())` | 可以先 new 留默认值，只 set 需要的 |
| **注解开销** | `@Data + @Builder + @NoArgsConstructor + @AllArgsConstructor` = 4 个 | `@Data + @Accessors(chain=true)` = 2 个 |
| **可读性** | Builder 和后续 setter 风格不统一 | 全部统一 `setXxx()` 链 |
| **运行时** | Builder 内部生成一个隐藏的 Builder 类，多一次对象创建 | 零额外开销，setter 返回 `this` |

### 3.4 适用性分析

**为什么适合本项目**：

1. AgentContext 是 mutable 的运行时上下文对象，不是值对象
2. 存在自引用初始化需求（`buildToolCollection` 依赖 `agentContext` 自身）
3. 运行时动态修改 context 的场景天然适配（`PlanSolveHandlerImpl.setSopPrompt()`）

**什么时候不适合**：

- 需要不可变语义时（build once, read-only）→ 用 Builder
- 字段极多且大多可选时 → Builder 的缺省跳过更方便

### 3.5 改造示例

```java
// Before
@Data
@Builder
@Slf4j
@NoArgsConstructor
@AllArgsConstructor
public class AgentContext { ... }

// After
@Data
@Accessors(chain = true)
@Slf4j
public class AgentContext {
    private String requestId;
    private String sessionId;
    private String query;
    private String task = "";
    private Printer printer;
    private ToolCollection toolCollection;
    private String dateInfo;
    private List<File> productFiles = new ArrayList<>();
    private Boolean isStream;
    private String streamMessageType;
    private String sopPrompt;
    private String basePrompt;
    private Integer agentType;
    private List<File> taskProductFiles = new ArrayList<>();
    private String templateType;
}
```

---

☕️ 制作不易，请我喝咖啡☕️关注我➕