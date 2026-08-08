# 05-06 Agent 系统 aibitat

> **所属章节**: [05-核心代码讲解](./05-核心代码讲解.md)  
> **核心文件**: `server/utils/agents/aibitat/index.js`、`server/utils/agents/aibitat/plugins/*`、`server/utils/agents/aibitat/providers/*`、`server/utils/agents/ephemeral.js`

---

## 1. 架构概述

**aibitat** 是 AnythingLLM 自研的 AI Agent 框架，名称来源于「AI + habit」（习惯），寓意 Agent 能够形成工具调用的习惯。它实现了完整的 **ReAct（Reasoning + Acting）** 模式。

### 1.1 核心概念

| 概念 | 说明 |
|------|------|
| **AIbitat** | Agent 引擎核心类，管理聊天历史、插件、执行循环 |
| **Plugin** | 工具插件，实现 `name`、`description`、`call()` 接口 |
| **Provider** | LLM Provider 适配，40+ Provider 均可用于 Agent |
| **Channel** | 通信通道（WebSocket、CLI、HTTP） |
| **EventEmitter** | 事件推送，用于实时通知前端 |

### 1.2 插件清单（20+）

| 插件 | 功能 | 文件 |
|------|------|------|
| web-browsing | 网页浏览（Puppeteer） | `plugins/web-browsing.js` |
| web-scraping | 网页内容抓取 | `plugins/web-scraping.js` |
| memory | 记忆系统访问 | `plugins/memory.js` |
| filesystem | 文件读写 | `plugins/filesystem/` |
| create-files | 文件创建 | `plugins/create-files/` |
| rechart | 图表生成（Recharts） | `plugins/rechart.js` |
| http-socket | HTTP 请求 | `plugins/http-socket.js` |
| gmail | Gmail 邮件操作 | `plugins/gmail/` |
| outlook | Outlook 邮件操作 | `plugins/outlook/` |
| google-calendar | Google 日历操作 | `plugins/google-calendar/` |
| sql-agent | SQL 查询执行 | `plugins/sql-agent/` |
| request-user-input | 请求用户输入 | `plugins/request-user-input.js` |
| summarize | 内容摘要 | `plugins/summarize.js` |
| cli | 命令行执行 | `plugins/cli.js` |
| chat-history | 聊天历史访问 | `plugins/chat-history.js` |
| file-history | 文件历史访问 | `plugins/file-history.js` |
| create-scheduled-job | 创建定时任务 | `plugins/create-scheduled-job/` |
| websocket | WebSocket 通信 | `plugins/websocket.js` |
| router-classifier | 工具智能选择 | `plugins/router-classifier.js` |

---

## 2. AIbitat 核心类

### 2.1 类属性

```javascript
class AIbitat {
  emitter = new EventEmitter();       // 事件发射器
  skipHandleExecution = false;        // 跳过执行标志
  _provider = null;                   // 当前 Provider 配置
  _providerInstance = null;           // Provider 实例
  defaultProvider = null;             // 默认 Provider
  defaultInterrupt;                   // 中断模式
  maxRounds = 100;                    // 最大循环轮次
  maxToolCalls = 10;                  // 最大工具调用次数
  _chats = [];                        // 聊天历史
  agents = new Map();                 // Agent 映射
  channels = new Map();               // 通道映射
  functions = new Map();              // 工具函数映射
  _pendingCitations = [];             // 待推送引用
  _toolAttachments = [];              // 待注入附件
  _pendingClarifyingQuestionSurveys = []; // 待处理问卷
}
```

### 2.2 构造函数

```javascript
constructor(props = {}) {
  const {
    chats = [],
    interrupt = "NEVER",
    maxRounds = 100,
    maxToolCalls = AIbitat.defaultMaxToolCalls(),
    provider = "openai",
    handlerProps = {},
    ...rest
  } = props;
  this._chats = chats;
  this.defaultInterrupt = interrupt;
  this.maxRounds = maxRounds;
  this.maxToolCalls = maxToolCalls;
  this.handlerProps = handlerProps;
  this.defaultProvider = { provider, ...rest };
  this.provider = this.defaultProvider.provider;
  this.model = this.defaultProvider.model;
}
```

**设计要点**：
- **配置驱动**：所有参数通过构造函数传入，便于测试与复用。
- **环境变量控制**：`maxToolCalls` 可通过 `AGENT_MAX_TOOL_CALLS` 环境变量覆盖。

### 2.3 静态方法

```javascript
static defaultMaxToolCalls() {
  const envMaxToolCalls = parseInt(process.env.AGENT_MAX_TOOL_CALLS, 10);
  return !isNaN(envMaxToolCalls) && envMaxToolCalls > 0
    ? envMaxToolCalls
    : 10;
}
```

---

## 3. ReAct 执行循环

### 3.1 主循环逻辑（简化伪代码）

```javascript
async function startAgentCluster() {
  let round = 0;
  let toolCallCount = 0;

  while (round < this.maxRounds) {
    round++;

    // 1. 调用 LLM 获取响应
    const response = await this.providerInstance.chatComplete(
      this._chats,
      { tools: this.getToolDefinitions() }
    );

    // 2. 解析响应
    if (!response.hasToolCalls) {
      // 无工具调用 → 最终回答
      this.emitter.emit("final", response.content);
      return { output: response.content };
    }

    // 3. 处理工具调用
    for (const toolCall of response.toolCalls) {
      if (toolCallCount >= this.maxToolCalls) {
        this.emitter.emit("warning", "达到工具调用上限");
        break;
      }
      toolCallCount++;

      // 4. 执行工具
      const plugin = this.functions.get(toolCall.name);
      if (!plugin) {
        this._chats.push({ role: "tool", content: `Unknown tool: ${toolCall.name}` });
        continue;
      }

      // 5. 推送工具调用事件
      this.emitter.emit("toolCall", { name: toolCall.name, params: toolCall.params });

      // 6. 执行并获取结果
      const result = await plugin.call(toolCall.params, this.handlerProps);

      // 7. 推送工具结果事件
      this.emitter.emit("toolResult", { name: toolCall.name, result });

      // 8. 将结果加入聊天历史
      this._chats.push({ role: "tool", content: JSON.stringify(result) });
    }
  }

  // 达到最大轮次
  this.emitter.emit("warning", "达到最大循环轮次");
}
```

### 3.2 工具定义生成

```javascript
getToolDefinitions() {
  return Array.from(this.functions.values()).map(plugin => ({
    type: "function",
    function: {
      name: plugin.name,
      description: plugin.description,
      parameters: plugin.parameters, // JSON Schema
    },
  }));
}
```

---

## 4. 插件接口

### 4.1 Plugin 接口契约

```javascript
// 每个插件需导出以下结构
module.exports = {
  name: "web-browsing",                    // 工具名称（唯一）
  description: "浏览网页并获取内容",        // 工具描述（供 LLM 理解）
  parameters: {                            // 参数 JSON Schema
    type: "object",
    properties: {
      url: { type: "string", description: "要浏览的 URL" },
    },
    required: ["url"],
  },
  call: async (params, handlerProps) => {  // 执行函数
    const { url } = params;
    // ... 执行逻辑
    return { output: "网页内容..." };
  },
};
```

### 4.2 Web Browsing 插件示例

```javascript
const puppeteer = require("puppeteer");

module.exports = {
  name: "web-browsing",
  description: "浏览网页并获取文本内容",
  parameters: {
    type: "object",
    properties: {
      url: { type: "string", description: "要浏览的 URL" },
    },
    required: ["url"],
  },
  async call({ url }) {
    const browser = await puppeteer.launch({ headless: true });
    const page = await browser.newPage();
    await page.goto(url, { waitUntil: "networkidle2" });
    const content = await page.evaluate(() => document.body.innerText);
    await browser.close();
    return { output: content.slice(0, 5000) }; // 截断避免 Token 爆炸
  },
};
```

---

## 5. EphemeralAgentHandler

### 5.1 功能

`EphemeralAgentHandler`（`server/utils/agents/ephemeral.js`）是一次性 Agent 执行器，用于：
- 定时任务中的 Agent 执行
- API 触发的 Agent 执行
- 无需持久会话的场景

### 5.2 与 SessionAgentHandler 的区别

| 特性 | EphemeralAgentHandler | SessionAgentHandler |
|------|----------------------|---------------------|
| 生命周期 | 单次执行 | 持久会话 |
| 通信方式 | 无 WebSocket | WebSocket 双向 |
| 状态保持 | 无 | 保持聊天历史 |
| 适用场景 | 定时任务、API | 交互式 Agent 聊天 |

### 5.3 核心方法

```javascript
class EphemeralAgentHandler extends AgentHandler {
  #invocationUUID = null;
  #workspace = null;
  #userId = null;
  #prompt = null;
  #funcsToLoad = [];

  async init() {
    this.aibitat = new AIbitat({
      provider: this.provider,
      model: this.model,
      chats: [],
      handlerProps: { workspace: this.#workspace, userId: this.#userId },
    });

    // 加载指定插件
    for (const funcName of this.#funcsToLoad) {
      const plugin = AgentPlugins.get(funcName);
      this.aibitat.addPlugin(plugin);
    }

    return this;
  }

  async run() {
    this.aibitat._chats.push({ role: "user", content: this.#prompt });
    const result = await this.aibitat.startAgentCluster();
    return result;
  }
}
```

---

## 6. 工具智能选择（router-classifier）

### 6.1 问题

传统 Agent 将所有工具描述注入 System Prompt，导致：
- Token 消耗大（20+ 工具描述可能占用 2000+ tokens）
- LLM 注意力分散，工具选择准确率下降

### 6.2 解决方案

`router-classifier.js` 插件在执行前对工具进行 **预分类**：

```javascript
async function classify(tools, userMessage) {
  // 使用轻量 LLM 调用对工具进行相关性评分
  const response = await llm.chatComplete([
    { role: "system", content: "对以下工具与用户问题的相关性评分（0-10）" },
    { role: "user", content: `问题: ${userMessage}\n工具: ${tools.map(t => t.name).join(", ")}` },
  ]);

  // 解析评分，保留 Top-K 工具
  const scores = parseScores(response);
  return tools.filter((_, i) => scores[i] >= THRESHOLD).slice(0, 5);
}
```

**效果**：官方宣称可减少 **80% Token 消耗**。

---

## 7. 潜在问题与改进建议

| 问题 | 影响 | 建议 |
|------|------|------|
| 无插件沙箱隔离 | 恶意插件可访问文件系统 | 使用 VM2 或 Worker 隔离 |
| 工具调用无超时 | 卡死导致资源泄漏 | 为每个工具调用添加超时 |
| 无调用链追踪 | 调试困难 | 添加结构化日志追踪每次工具调用 |
| maxRounds 硬编码 | 不同任务需要不同轮次 | 根据任务复杂度动态调整 |
| 插件加载无版本管理 | 插件兼容性问题 | 添加插件版本声明与校验 |

---

**返回 → [05-核心代码讲解.md](./05-核心代码讲解.md)** | **上一子文件 ← [05-05-聊天引擎与RAG管线.md](./05-05-聊天引擎与RAG管线.md)** | **下一子文件 → [05-07-MCP集成.md](./05-07-MCP集成.md)**

---

# ❤️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../res/promotion.jpg)