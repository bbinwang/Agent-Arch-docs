# 第十部分：额外增强内容

## 10.1 代码走读文档（Code Walkthrough）×5

### 10.1.1 pi-ai 包走读

**架构**：pi-ai 是 Pi 的 LLM API 层，采用**提供者模式（Provider Pattern）**。核心抽象是 `Models`（Provider 集合）和 `Provider`（单个 LLM 提供商）。

**入口**：`src/index.ts` 导出核心类型和函数。

**核心流程**：
1. 创建 `Models` 实例
2. 添加 Provider（`models.addProvider(openaiProvider())`）
3. 选择 Model
4. 调用 `streamSimple(model, context, options)`
5. 处理事件流

**关键类型**：
- `Provider<TApi>`：provider 接口
- `Model<TApi>`：模型定义
- `AssistantMessageEventStream`：事件流
- `Credential`：凭证（apiKey / oauth）

**扩展点**：
- 自定义 Provider：实现 `Provider` 接口
- 自定义 API：在 `api/` 目录添加新实现

---

### 10.1.2 pi-agent-core 包走读

**架构**：pi-agent-core 是通用的 agent 运行时，采用**事件驱动架构**。

**入口**：`src/index.ts` 导出 Agent、agentLoop、compaction 等。

**核心流程**：
1. 创建 `Agent` 实例
2. 调用 `agentLoop(prompts, context, config)`
3. 订阅事件流
4. 处理 agent_end 事件

**关键类型**：
- `AgentState`：agent 状态
- `AgentContext`：agent 上下文
- `AgentEvent`：事件类型
- `AgentTool`：工具接口

**扩展点**：
- 自定义 AgentLoopConfig：hooks、transformContext 等
- 自定义 AgentTool：实现 Tool 接口

---

### 10.1.3 pi-coding-agent 包走读

**架构**：pi-coding-agent 是应用层，采用**分层架构 + 插件架构**。

**入口**：
- CLI：`src/main.ts` → `main()`
- SDK：`src/index.ts` → `createAgentSession()`
- RPC：`src/rpc-entry.ts`

**核心流程**：
1. 解析参数
2. 创建 SessionManager
3. 创建 AgentSessionRuntime
4. 分发到运行模式

**关键类**：
- `AgentSession`：会话核心
- `SessionManager`：会话持久化
- `SettingsManager`：设置管理
- `ExtensionRunner`：扩展运行

**扩展点**：
- 自定义工具：`createXxxTool()`
- 自定义扩展：Extension API
- 自定义模式：继承 AgentSession

---

### 10.1.4 pi-tui 包走读

**架构**：pi-tui 是终端 UI 库，采用**差分渲染架构**。

**入口**：`src/index.ts` 导出 TUI 引擎和组件。

**核心概念**：
- 双缓冲区：当前帧 + 上一帧
- 差异检测：计算帧间差异
- 区域更新：只更新变化区域

**关键类**：
- `TuiEngine`：TUI 引擎
- `Editor`：编辑器组件
- `Markdown`：Markdown 渲染器
- `SelectList`：选择列表

**原生模块**：
- macOS：`darwin-modifiers.c` — 终端修饰键
- Windows：`win32-console-mode.c` — 控制台模式

---

### 10.1.5 pi-orchestrator 包走读

**架构**：pi-orchestrator 是编排器，采用**中介者模式**。

**入口**：`src/index.ts` 导出 supervisor、serve、storage 等。

**核心流程**：
1. 启动 HTTP serve
2. 接收创建实例请求
3. 启动 `pi --rpc` 子进程
4. 管理实例生命周期
5. 转发命令和事件

**关键类**：
- `OrchestratorSupervisor`：实例管理
- `RpcProcess`：RPC 子进程
- `serve`：HTTP 服务

---

## 10.2 开发者上手指南

### 本地运行

```bash
# 1. 克隆仓库
git clone https://github.com/earendil-works/pi.git
cd pi

# 2. 安装依赖（不执行 lifecycle 脚本）
npm install --ignore-scripts

# 3. 构建所有包
npm run build

# 4. 运行 Pi（从源码）
./pi-test.sh

# 5. 或直接运行
node packages/coding-agent/dist/cli.js
```

### 调试

#### VS Code Launch 配置

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Debug Pi",
      "type": "node",
      "request": "launch",
      "runtimeExecutable": "${workspaceFolder}/node_modules/.bin/tsx",
      "args": ["packages/coding-agent/src/main.ts", "--interactive"],
      "cwd": "${workspaceFolder}",
      "env": {
        "OPENAI_API_KEY": "your-key"
      }
    }
  ]
}
```

#### tmux 自动化测试

```bash
# 启动 tmux 会话
tmux new-session -d -s pi-test -x 80 -y 24

# 运行 Pi
tmux send-keys -t pi-test "./pi-test.sh" Enter

# 等待启动
sleep 3
tmux capture-pane -t pi-test -p

# 发送 prompt
tmux send-keys -t pi-test "Say exactly: ok" Enter

# 等待响应
sleep 5
tmux capture-pane -t pi-test -p

# 清理
tmux kill-session -t pi-test
```

### 测试

```bash
# 运行所有测试（跳过需要 API key 的 e2e 测试）
./test.sh

# 运行特定包的测试
cd packages/coding-agent && npm test

# 运行特定测试文件
node ../../node_modules/vitest/dist/cli.js --run test/session-manager/session.test.ts

# 运行 harness 测试（agent-core）
cd packages/agent && npm run test:harness
```

### 贡献流程

1. Fork 仓库
2. 创建功能分支
3. 编写代码和测试
4. 运行 `npm run check` 确保通过
5. 提交（遵循 commit message 格式）
6. 创建 PR
7. 等待审查

**Commit Message 格式**：
```
{feat,fix,docs}[(ai,tui,agent,coding-agent)]: <message>
```

**代码规则**（详见 AGENTS.md）：
- 无 `any`（除非绝对必要）
- 无 inline imports
- 使用 erasable TypeScript syntax
- 读取完整文件后再修改
- 不要运行 `npm run build` 或 `npm test`（除非请求）

---

## 10.3 架构决策记录（ADR）

### ADR-001：为什么使用 Monorepo

**状态**：已接受

**背景**：Pi 包含多个相互关联的包，需要统一管理和发布。

**决策**：使用 npm workspaces monorepo。

**理由**：
- 原子提交：跨包变更一次提交
- 共享工具链：Biome、TypeScript 配置统一
- lockstep 版本：所有包共享版本号
- 本地开发：修改一个包后其他包立即可见

**后果**：
- 构建时间较长（需构建所有包）
- 发布需要协调

---

### ADR-002：为什么使用 JSONL 会话格式

**状态**：已接受

**背景**：需要持久化会话历史，支持崩溃恢复和分叉。

**决策**：使用 JSONL（每行一个 JSON 对象）。

**理由**：
- 追加写入：新消息追加到末尾，崩溃安全
- 人类可读：可以直接查看和编辑
- 可分叉：复制文件即可创建分支
- 可审计：完整历史记录

**后果**：
- 大文件读取慢（需要 compaction）
- 无查询能力（需要全量扫描）

---

### ADR-003：为什么分离 ai / agent-core / coding-agent

**状态**：已接受

**背景**：需要平衡复用性和特定性。

**决策**：三层分离。

**理由**：
- pi-ai 可独立用于任何 LLM 调用
- pi-agent-core 可独立用于非编码 agent
- pi-coding-agent 专注编码特定功能

**后果**：
- 模块边界清晰
- 独立测试和发布

---

### ADR-004：为什么自扩展设计

**状态**：已接受

**背景**：核心功能稳定，但用户需求多样。

**决策**：提供 Extension API，允许用户扩展。

**理由**：
- 开放-封闭原则
- 社区生态
- 领域隔离

**后果**：
- 扩展质量不可控
- API 维护成本

---

### ADR-005：为什么事件驱动 Agent Loop

**状态**：已接受

**背景**：需要流式 UI 更新和解耦。

**决策**：Agent loop 通过事件流通信。

**理由**：
- 流式 UI 更新
- 解耦 agent 和 UI
- 扩展钩子

**后果**：
- 事件类型复杂
- 调试困难

---

### ADR-006：为什么 Bun + Node 双运行时

**状态**：已接受

**背景**：需要兼顾性能和兼容性。

**决策**：同时支持 Node.js 和 Bun。

**理由**：
- Bun：单二进制、启动快
- Node.js：兼容性广、生态成熟

**后果**：
- 双份构建/测试
- 运行时差异需要处理

---

## 10.4 关键算法分析

### 10.4.1 上下文压缩算法（findCutPoint）

**功能**：找到最佳压缩点，保留最近完整轮次。

**伪代码**：
```
function findCutPoint(messages):
  // 从末尾开始扫描
  for i from messages.length-1 downto 0:
    msg = messages[i]

    // 找到新会话或压缩标记
    if msg.type == "new_session" or msg.type == "compaction":
      return { cutIndex: i, keepAfter: messages.length - i }

    // 确保不切断工具调用
    if msg.type == "assistant" and hasToolCalls(msg):
      // 找到对应的 tool_result
      toolResults = findToolResults(messages, msg)
      if all toolResults exist:
        return { cutIndex: i, keepAfter: messages.length - i }

  // 默认：压缩一半
  return { cutIndex: messages.length / 2, keepAfter: messages.length / 2 }
```

**复杂度**：
- 时间：O(n)，n 为消息数
- 空间：O(1)

**优化建议**：
- 使用二分查找加速（如果消息按轮次分组）
- 缓存轮次边界

---

### 10.4.2 差分渲染算法

**功能**：计算两帧差异，只更新变化区域。

**伪代码**：
```
function diffRender(currentFrame, previousFrame):
  changes = []

  // 逐行比较
  for y from 0 to terminalHeight:
    currentLine = currentFrame.getLine(y)
    previousLine = previousFrame.getLine(y)

    if currentLine != previousLine:
      // 逐字符比较
      for x from 0 to terminalWidth:
        if currentLine[x] != previousLine[x]:
          changes.push({ x, y, char: currentLine[x] })

  // 批量输出变化
  outputChanges(changes)
```

**复杂度**：
- 时间：O(W×H)，W 为宽度，H 为高度
- 空间：O(C)，C 为变化数

**优化建议**：
- 使用脏矩形（dirty rectangle）减少比较区域
- 使用哈希加速行比较

---

### 10.4.3 Token 估算算法

**功能**：估算消息列表的 token 数。

**伪代码**：
```
function estimateTokens(messages):
  total = 0

  for msg in messages:
    // 基础 token（角色、结构）
    total += 4

    // 内容 token
    if msg.content is string:
      total += estimateStringTokens(msg.content)
    else:
      for block in msg.content:
        if block.type == "text":
          total += estimateStringTokens(block.text)
        else if block.type == "toolCall":
          total += estimateStringTokens(block.name)
          total += estimateStringTokens(JSON.stringify(block.arguments))

    // 工具结果
    if msg.type == "toolResult":
      total += estimateStringTokens(msg.content)

  return total

function estimateStringTokens(str):
  // 简化估算：英文约 4 字符/token，中文约 1.5 字符/token
  return ceil(str.length / 3)
```

**复杂度**：
- 时间：O(N)，N 为总字符数
- 空间：O(1)

**优化建议**：
- 使用模型的 tokenizer（如 tiktoken）获得精确计数
- 缓存常见字符串的 token 数

---

### 10.4.4 Fuzzy 匹配算法（model-resolver）

**功能**：根据 pattern 匹配模型 ID。

**伪代码**：
```
function fuzzyMatch(pattern, modelId):
  // 精确匹配
  if pattern == modelId:
    return 1.0

  // 前缀匹配
  if modelId.startsWith(pattern):
    return 0.9

  // 包含匹配
  if modelId.includes(pattern):
    return 0.8

  // 模糊匹配（编辑距离）
  distance = levenshteinDistance(pattern, modelId)
  similarity = 1 - distance / max(pattern.length, modelId.length)

  return similarity
```

**复杂度**：
- 时间：O(P×M)，P 为 pattern 长度，M 为 modelId 长度
- 空间：O(P×M)

**优化建议**：
- 使用 Trie 树加速前缀匹配
- 使用 BK-tree 加速模糊匹配

---

## 10.5 测试策略与主要测试用例

### 测试层级

```
┌─────────────────────────────────────────┐
│  E2E 测试（需要 API key）                │
│  真实 provider 调用，验证完整流程          │
├─────────────────────────────────────────┤
│  集成测试（suite/ + faux provider）       │
│  模拟 provider，验证多组件协作             │
├─────────────────────────────────────────┤
│  单元测试（各模块 .test.ts）              │
│  独立函数和类的测试                       │
└─────────────────────────────────────────┘
```

### 单元测试

**目标**：独立函数和类的测试。

**示例**：
```typescript
// session-manager.test.ts
describe("SessionManager", () => {
  it("should create a new session", async () => {
    const sm = SessionManager.create("/tmp/test");
    expect(sm).toBeDefined();
    expect(sm.getCwd()).toBe("/tmp/test");
  });

  it("should append entries", async () => {
    const sm = SessionManager.create("/tmp/test");
    sm.appendEntry({ type: "message", message: { role: "user", content: "hi" } });
    const context = sm.buildSessionContext();
    expect(context.messages.length).toBe(1);
  });
});
```

### 集成测试

**目标**：多组件协作测试，使用 faux provider 避免真实 API 调用。

**示例**：
```typescript
// suite/agent-session.test.ts
import { createAgentSession } from "../../src/core/sdk.ts";
import { fauxProvider } from "@earendil-works/pi-ai/providers/faux.ts";

describe("AgentSession integration", () => {
  it("should run a full turn with tools", async () => {
    const result = await createAgentSession({
      cwd: "/tmp/test",
      model: fauxProvider.models[0],
      tools: ["bash", "read"],
    });

    const messages = await result.session.prompt("List files");
    expect(messages.length).toBeGreaterThan(0);
  });
});
```

### 回归测试

**目标**：防止已修复的 bug 再次出现。

**位置**：`packages/coding-agent/test/suite/regressions/`

**命名**：`<issue-number>-<short-slug>.test.ts`

**示例**：
```typescript
// regressions/6695-all-argument-defaults.test.ts
describe("Issue #6695: all-argument prompt defaults", () => {
  it("should support all-argument prompt defaults", async () => {
    // 测试代码
  });
});
```

### E2E 测试

**目标**：真实 provider 调用，验证完整流程。

**触发**：当环境变量包含 provider API key 时自动激活。

**注意**：本地运行 `./test.sh` 会跳过 E2E 测试（无 API key）。

### 烟雾测试

**目标**：发布前的最终验证。

**位置**：`npm run release:local`

**内容**：
- Node 包安装测试
- Bun 二进制测试
- 启动测试
- 交互模式测试
- 真实 prompt 测试

### 覆盖矩阵

| 模块 | 单元测试 | 集成测试 | 回归测试 | E2E |
|------|---------|---------|---------|-----|
| agent-session | ✅ | ✅ | ✅ | ❌ |
| session-manager | ✅ | ✅ | ✅ | ❌ |
| tools | ✅ | ✅ | ✅ | ❌ |
| compaction | ✅ | ✅ | ✅ | ❌ |
| extensions | ✅ | ✅ | ✅ | ❌ |
| model-resolver | ✅ | ❌ | ❌ | ❌ |
| interactive | ❌ | ✅ | ✅ | ❌ |
| rpc | ❌ | ✅ | ✅ | ❌ |

---

## 10.6 术语表（Glossary）

| 术语 | 定义 |
|------|------|
| **AgentSession** | Pi 的核心类，封装 agent 生命周期（会话、工具、事件、compaction） |
| **AgentLoop** | 核心循环，驱动 LLM 调用和工具执行 |
| **Compaction** | 上下文压缩，将历史消息替换为摘要以节省 token |
| **Extension** | 扩展，通过 Extension API 添加自定义能力 |
| **ScopedModel** | 作用域模型，用于 Ctrl+P 循环切换的模型列表 |
| **ThinkingLevel** | 思考级别（off/minimal/low/medium/high/xhigh/max） |
| **ProjectTrust** | 项目信任，控制扩展对项目资源的访问 |
| **SessionManager** | 会话管理器，负责 JSONL 持久化 |
| **SettingsManager** | 设置管理器，三级设置（全局/项目/会话） |
| **ModelRuntime** | 模型运行时，管理可用模型和认证 |
| **ResourceLoader** | 资源加载器，加载扩展/skills/templates/themes |
| **ExtensionRunner** | 扩展运行器，分发事件到扩展钩子 |
| **Provider** | LLM 提供商（如 OpenAI、Anthropic） |
| **Model** | 模型定义（包含 provider、id、api、capabilities） |
| **StreamSimple** | 简化的流式调用接口 |
| **EventStream** | 事件流，agent loop 的输出 |
| **RpcMode** | RPC 模式，JSONL over stdio |
| **InteractiveMode** | 交互模式，全屏 TUI |
| **PrintMode** | 打印模式，非交互文本输出 |
| **FauxProvider** | 假 provider，用于测试 |
| **Shrinkwrap** | 传递依赖锁定文件 |
| **Lockstep** | 所有包共享版本号 |
| **UUIDv7** | 时间有序 UUID，用作 sessionId |
| **Jiti** | 运行时 TypeScript 执行器，用于加载扩展 |
| **TypeBox** | 类型安全的 schema 库，用于工具参数验证 |
| **Photon** | Rust WASM 图像处理库 |
| **Biome** | 代码检查和格式化工具 |
| **tsgo** | TypeScript 原生编译器（Node strip-only 模式） |
| **Gondolin** | 本地 Linux micro-VM 沙箱 |
| **Orchestrator** | 多实例编排器 |
| **Radius** | 存在/发现服务（orchestrator 中） |
| **PKCE** | Proof Key for Code Exchange，OAuth 安全扩展 |
| **Device Code** | OAuth 设备码流程 |
| **LSP** | Language Server Protocol，语言服务器协议 |

---

☕️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../res/promotion.jpg)