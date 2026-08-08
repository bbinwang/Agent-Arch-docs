# 第七部分：API 与接口设计

## 7.1 API 总览

Pi 提供三类对外接口：

| 接口类型 | 入口 | 协议 | 使用场景 |
|---------|------|------|---------|
| CLI | `pi` 命令 | 命令行参数 + stdin/stdout | 终端使用 |
| SDK | `createAgentSession()` | TypeScript API | 程序化集成 |
| RPC | `pi --rpc` | JSONL over stdio | IDE 插件、自动化 |

---

## 7.2 CLI 接口

### 参数列表

| 参数 | 类型 | 说明 |
|------|------|------|
| `--model` | string | 模型标识，支持 `provider/model` 或 `model` |
| `--provider` | string | 提供商标识 |
| `--thinking` | string | 思考级别：off/minimal/low/medium/high/xhigh/max |
| `--api-key` | string | API Key（运行时覆盖） |
| `--session` | string | 打开指定会话（ID 或路径） |
| `--continue` | flag | 继续最近会话 |
| `--resume` | flag | 交互式选择会话 |
| `--fork` | string | 分叉指定会话 |
| `--no-session` | flag | 不持久化会话 |
| `--session-id` | string | 指定会话 ID |
| `--name` | string | 会话名称 |
| `--print` / `-p` | flag/string | 非交互模式 |
| `--rpc` | flag | RPC 模式 |
| `--json` | flag | JSON 输出模式 |
| `--messages` | string[] | 初始消息 |
| `--file` / `@file` | string | 从文件读取输入 |
| `--image` | string | 输入图像 |
| `--tools` | string[] | 工具白名单 |
| `--exclude-tools` | string[] | 工具黑名单 |
| `--no-tools` | flag | 禁用所有工具 |
| `--no-builtin-tools` | flag | 禁用内置工具 |
| `--extensions` | string[] | 加载扩展 |
| `--no-extensions` | flag | 不加载扩展 |
| `--skills` | string[] | 加载 skills |
| `--prompt-templates` | string[] | 加载 prompt templates |
| `--themes` | string[] | 加载主题 |
| `--system-prompt` | string | 覆盖 system prompt |
| `--append-system-prompt` | string | 追加 system prompt |
| `--offline` | flag | 离线模式 |
| `--version` / `-v` | flag | 显示版本 |
| `--help` / `-h` | flag | 显示帮助 |
| `--list-models` | flag/string | 列出可用模型 |
| `--export` | string | 导出会话为 HTML |
| `--config` | flag | 配置模式 |
| `--project-trust` | boolean | 覆盖项目信任 |
| `--verbose` | flag | 详细输出 |

### 退出码

| 码 | 含义 |
|----|------|
| 0 | 成功 |
| 1 | 一般错误（参数错误、模型不可用等） |
| 其他 | 工具执行失败等 |

### stdin/stdout 语义

- **交互模式**：stdin 为 TTY，进入全屏 TUI
- **管道输入**：`echo "..." | pi` → stdin 非 TTY，自动进入 print 模式
- **JSON 模式**：`pi --json` → stdout 输出结构化 JSON
- **RPC 模式**：stdin/stdout 用于 JSONL 协议通信

---

## 7.3 SDK 接口

### createAgentSession

```typescript
import { createAgentSession } from "@earendil-works/pi-coding-agent";

const result = await createAgentSession({
  cwd: "/path/to/project",
  model: myModel,
  thinkingLevel: "medium",
  scopedModels: [
    { model: model1, thinkingLevel: "low" },
    { model: model2, thinkingLevel: "high" },
  ],
});

const { session, extensionsResult, modelFallbackMessage } = result;

// 发送消息
const messages = await session.prompt("Fix the bug in src/main.ts");

// 订阅事件
const unsubscribe = session.subscribe((event) => {
  console.log(event);
});

// 清理
unsubscribe();
```

### createAgentSessionRuntime

```typescript
import { createAgentSessionRuntime } from "@earendil-works/pi-coding-agent";

const runtime = await createAgentSessionRuntime(createRuntime, {
  cwd: "/path/to/project",
  agentDir: "~/.pi/agent",
  sessionManager,
});

// 运行时支持 session 切换后重建
const { session, services, diagnostics } = runtime;
```

### 工具工厂

```typescript
import {
  createBashTool,
  createCodingTools,
  createReadOnlyTools,
  createEditTool,
  createReadTool,
  createWriteTool,
  createFindTool,
  createGrepTool,
  createLsTool,
} from "@earendil-works/pi-coding-agent";

const bashTool = createBashTool();
const codingTools = createCodingTools();
const readOnlyTools = createReadOnlyTools();
```

### 运行模式

```typescript
import { InteractiveMode, runPrintMode, runRpcMode } from "@earendil-works/pi-coding-agent";

// 交互模式
const interactive = new InteractiveMode(runtime, options);
await interactive.run();

// 打印模式
const exitCode = await runPrintMode(runtime, {
  mode: "text",
  initialMessage: "...",
});

// RPC 模式
await runRpcMode(runtime);
```

---

## 7.4 RPC 协议

### 协议帧格式

每行一个 JSON 对象：

```jsonl
{"type":"initialize"}
{"type":"initialized"}
{"type":"prompt","id":"req_1","message":"Fix the bug"}
{"type":"response","id":"req_1","success":true,"data":{...}}
{"type":"event","event":{"type":"agent_start"}}
{"type":"event","event":{"type":"message_update","message":{...}}}
```

### 命令类型

#### prompt
```json
{
  "type": "prompt",
  "id": "req_1",
  "message": "Fix the bug in src/main.ts",
  "images": [{"type": "image", "data": "base64...", "mediaType": "image/png"}]
}
```

#### get_state
```json
{
  "type": "get_state"
}
```

#### new_session
```json
{
  "type": "new_session",
  "cwd": "/path/to/project"
}
```

#### switch_session
```json
{
  "type": "switch_session",
  "sessionFile": "~/.pi/sessions/xxx.jsonl"
}
```

#### fork
```json
{
  "type": "fork",
  "sourceSessionFile": "~/.pi/sessions/xxx.jsonl"
}
```

#### set_session_name
```json
{
  "type": "set_session_name",
  "name": "Fix authentication bug"
}
```

### 响应类型

#### 成功响应
```json
{
  "type": "response",
  "id": "req_1",
  "success": true,
  "data": {
    "sessionId": "0197c2e0-...",
    "sessionFile": "~/.pi/sessions/0197c2e0-....jsonl"
  }
}
```

#### 错误响应
```json
{
  "type": "response",
  "id": "req_1",
  "success": false,
  "error": "No model available"
}
```

### 事件流

```json
{"type":"event","event":{"type":"agent_start"}}
{"type":"event","event":{"type":"turn_start"}}
{"type":"event","event":{"type":"message_start","message":{...}}}
{"type":"event","event":{"type":"message_update","message":{...},"assistantMessageEvent":{...}}}
{"type":"event","event":{"type":"message_end","message":{...}}}
{"type":"event","event":{"type":"tool_execution_start","toolCallId":"...","toolName":"bash","args":{...}}}
{"type":"event","event":{"type":"tool_execution_end","toolCallId":"...","toolName":"bash","result":{...},"isError":false}}
{"type":"event","event":{"type":"turn_end","message":{...},"toolResults":[{...}]}}
{"type":"event","event":{"type":"agent_end","messages":[{...}]}}
```

### UI 请求/响应桥接

Pi 可以通过 RPC 请求外部进程显示 UI（如选择器）：

```json
// Pi → 外部
{
  "type": "ui_request",
  "id": "ui_1",
  "uiType": "select",
  "title": "Choose model",
  "options": [
    {"label": "GPT-4o", "value": "openai/gpt-4o"},
    {"label": "Claude Sonnet", "value": "anthropic/claude-sonnet-4-20250514"}
  ]
}

// 外部 → Pi
{
  "type": "ui_response",
  "id": "ui_1",
  "value": "openai/gpt-4o"
}
```

---

## 7.5 扩展 API

### Extension 接口

```typescript
interface Extension {
  name: string;
  description?: string;

  // Slash 命令
  commands?: ExtensionCommand[];

  // 工具
  tools?: RegisteredTool[];

  // 事件钩子
  events?: {
    beforeAgentStart?: (event: BeforeAgentStartEvent) => Promise<BeforeAgentStartEventResult | void>;
    agentStart?: (event: AgentStartEvent) => Promise<void>;
    agentEnd?: (event: AgentEndEvent) => Promise<void>;
    turnStart?: (event: TurnStartEvent) => Promise<void>;
    turnEnd?: (event: TurnEndEvent) => Promise<void>;
    beforeToolCall?: (event: BeforeToolCallEvent) => Promise<ToolCallEventResult | void>;
    afterToolCall?: (event: AfterToolCallEvent) => Promise<void>;
    onMessage?: (event: MessageEvent) => Promise<void>;
    onInput?: (event: InputEvent) => Promise<InputEventResult | void>;
    beforeProviderRequest?: (event: BeforeProviderRequestEvent) => Promise<BeforeProviderRequestEventResult | void>;
    beforeProviderHeaders?: (event: BeforeProviderHeadersEvent) => Promise<void>;
    sessionBeforeCompact?: (event: SessionBeforeCompactEvent) => Promise<SessionBeforeCompactResult | void>;
    sessionBeforeFork?: (event: SessionBeforeForkEvent) => Promise<SessionBeforeForkResult | void>;
    sessionBeforeSwitch?: (event: SessionBeforeSwitchEvent) => Promise<SessionBeforeSwitchResult | void>;
    sessionShutdown?: (event: SessionShutdownEvent) => Promise<void>;
    // 更多...
  };

  // UI
  ui?: {
    widgets?: WidgetDefinition[];
    messageRenderers?: MessageRenderer[];
    entryRenderers?: EntryRenderer[];
    autocompleteProviders?: AutocompleteProviderFactory[];
  };
}
```

### defineTool

```typescript
import { defineTool } from "@earendil-works/pi-coding-agent";
import { Type } from "typebox";

const myTool = defineTool({
  name: "my_tool",
  description: "Does something useful",
  parameters: Type.Object({
    input: Type.String(),
  }),
  execute: async (toolCallId, params, signal) => {
    return {
      content: [{ type: "text", text: `Processed: ${params.input}` }],
      details: { processed: true },
    };
  },
});
```

### Extension 示例

```typescript
import type { ExtensionFactory } from "@earendil-works/pi-coding-agent";

const myExtension: ExtensionFactory = (context) => ({
  name: "my-extension",
  description: "An example extension",

  commands: [
    {
      name: "/greet",
      description: "Say hello",
      execute: async (args) => {
        return `Hello, ${args || "world"}!`;
      },
    },
  ],

  tools: [myTool],

  events: {
    beforeToolCall: async (event) => {
      if (event.toolName === "bash" && event.args.command.includes("rm -rf")) {
        return { block: true, reason: "Dangerous command blocked" };
      }
    },

    afterToolCall: async (event) => {
      context.logger.info(`Tool ${event.toolName} executed`);
    },
  },
});

export default myExtension;
```

---

## 7.6 Provider API（内部）

### Provider 接口

```typescript
interface Provider<TApi extends Api = Api> {
  readonly id: string;
  readonly name: string;
  readonly baseUrl?: string;
  readonly headers?: ProviderHeaders;

  // 认证
  auth: ProviderAuth;

  // 模型列表
  models: Model[];
  refreshModels(options?: RefreshModelsContext): Promise<void>;

  // 流式调用
  stream(
    model: Model<TApi>,
    context: Context,
    options?: StreamOptions,
  ): AssistantMessageEventStream;
}
```

### 创建自定义 Provider

```typescript
import type { Provider } from "@earendil-works/pi-ai";

const myProvider: Provider<"openai-responses"> = {
  id: "my-provider",
  name: "My Provider",
  baseUrl: "https://api.example.com/v1",
  auth: { type: "apiKey", envVar: "MY_PROVIDER_API_KEY" },
  models: [
    {
      id: "my-model",
      name: "My Model",
      api: "openai-responses",
      contextWindow: 128000,
      capabilities: { toolCalling: true, vision: true },
    },
  ],
  async refreshModels() { /* 刷新模型列表 */ },
  stream(model, context, options) { /* 实现流式调用 */ },
};
```

---

☕️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../res/promotion.jpg)