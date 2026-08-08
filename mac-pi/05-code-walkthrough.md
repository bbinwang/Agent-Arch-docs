# 第五部分：核心代码讲解（逐文件、逐函数深度走读）

## 5.1 coding-agent/src/main.ts — 入口与启动

### 文件概述
`main.ts` 是 Pi CLI 的主入口文件，包含 `main()` 函数。它负责将 CLI 参数转换为运行时配置，创建会话和运行时，然后分发到具体的运行模式。

### 核心函数

#### `main(args: string[], options?: MainOptions)`
**位置**：`main.ts:473`
**功能**：主入口函数

**执行流程**：
1. **计时重置**：`resetTimings()` — 为性能分析准备
2. **扩展工厂合并**：`extensionFactories = [...builtInExtensions, ...(options?.extensionFactories ?? [])]` — 合并内置扩展和外部传入的扩展
3. **离线模式检测**：检查 `--offline` 参数或 `PI_OFFLINE` 环境变量
4. **Windows 自更新清理**：`cleanupWindowsSelfUpdateQuarantine()` — Windows 特定处理
5. **HTTP 代理配置**：`applyHttpProxySettings()` + `configureHttpDispatcher()`
6. **包命令处理**：`handlePackageCommand()` — 处理 `pi update` 等命令
7. **配置命令处理**：`handleConfigCommand()` — 处理 `pi config` 命令
8. **参数解析**：`parseArgs(args)` — 解析为结构化 `Args`
9. **快速路径**：`--version`、`--export`、`--help`、`--list-models`
10. **模式确定**：`resolveAppMode()` — interactive/print/rpc
11. **迁移**：`runMigrations()` — 数据迁移
12. **设置管理**：`SettingsManager.create()` — 创建设置管理器
13. **首次设置**：`showFirstTimeSetup()` — 首次运行设置
14. **会话创建**：`createSessionManager()` — 创建/恢复会话
15. **运行时创建**：`createAgentSessionRuntime()` — 创建完整运行时
16. **输入准备**：`readPipedStdin()` + `prepareInitialMessage()`
17. **主题初始化**：`initTheme()`
18. **模式分发**：根据 `appMode` 运行对应模式

**设计模式**：
- **模板方法**：`main()` 定义了启动的固定步骤，某些步骤（如首次设置）是可选的
- **策略模式**：`appMode` 决定运行哪种模式
- **工厂模式**：`createAgentSessionRuntime` 是一个工厂函数

**潜在问题**：
- 函数过长（~400 行），包含大量逻辑，可考虑拆分
- 多个 `time()` 调用散落在代码中，是性能计价的侵入式标记

#### `resolveAppMode(parsed: Args, stdinIsTTY: boolean, stdoutIsTTY: boolean): AppMode`
**位置**：`main.ts:100`
**功能**：确定运行模式

**逻辑**：
- `rpc` 参数 → `"rpc"`
- `json` 参数 → `"json"`
- `print` 参数 或 stdin/stdout 非 TTY → `"print"`
- 其他 → `"interactive"`

#### `createSessionManager(parsed, cwd, sessionDir, settingsManager)`
**位置**：`main.ts:264`
**功能**：根据参数创建或恢复会话

**分支逻辑**：
- `--no-session` → 内存会话（不持久化）
- `--fork` → 分叉会话
- `--session` → 打开指定会话（支持路径或 ID 前缀匹配）
- `--resume` → 交互式选择会话
- `--continue` → 继续最近会话
- `--session-id` → 按 ID 打开或创建
- 默认 → 创建新会话

**异常处理**：
- 会话未找到：输出错误并 `process.exit(1)`
- 分叉时 ID 冲突：输出错误并退出

#### `buildSessionOptions(parsed, scopedModels, hasExistingSession, modelRuntime, settingsManager)`
**位置**：`main.ts:357`
**功能**：构建会话选项

**逻辑**：
- 从 CLI 参数解析模型（`--model`、`--provider`）
- 如果没有 CLI 模型，使用保存的默认模型或第一个 scoped model
- 解析 thinking level
- 构建 scoped models 列表（用于 Ctrl+P 循环）
- 处理工具列表（`--tools`、`--exclude-tools`、`--no-tools`）

---

## 5.2 coding-agent/src/core/agent-session.ts — 会话核心

### 文件概述
`agent-session.ts` 是 Pi 的核心文件，定义了 `AgentSession` 类。这个类是所有运行模式的共享抽象，封装了 agent 的完整生命周期。

### 核心类

#### `AgentSession`
**位置**：`agent-session.ts`
**职责**：会话核心，管理工具、事件、compaction、模型切换

**关键属性**：
- `agent`：底层 Agent 实例（来自 pi-agent-core）
- `sessionManager`：会话管理器
- `settingsManager`：设置管理器
- `modelRuntime`：模型运行时
- `resourceLoader`：资源加载器
- `extensionRunner`：扩展运行器
- `model`：当前模型
- `thinkingLevel`：当前思考级别

**关键方法**：

##### `prompt(message: string, images?: ImageContent[]): Promise<AgentMessage[]>`
发送用户消息并驱动 agent loop 直到完成。
1. 追加 user message entry 到会话
2. 调用 `agent.agentLoop()` 启动循环
3. 收集并返回所有新消息

##### `continue(): Promise<AgentMessage[]>`
从当前上下文继续（无新消息），用于重试。

##### `compact(options?: CompactOptions): Promise<CompactionResult>`
手动触发上下文压缩。
1. `prepareCompaction()` 准备压缩
2. `generateSummary()` 生成摘要
3. 写入 compaction entry
4. 返回压缩结果

##### `switchSession(path: string): Promise<void>`
切换到另一个会话。
1. 关闭当前会话
2. 打开新会话
3. 重建 agent 上下文

##### `forkSession(path: string): Promise<void>`
分叉当前会话。
1. 复制会话文件
2. 创建新会话

##### `setModel(model: Model<any>): void`
切换模型。
1. 更新 `this.model`
2. 发射 model_change event

##### `setThinkingLevel(level: ThinkingLevel): void`
设置思考级别。
1. 更新 `this.thinkingLevel`
2. 发射 thinking_level_change event

##### `subscribe(listener): Unsubscribe`
订阅 agent 事件。
1. 包装 listener，自动持久化事件到会话
2. 返回取消订阅函数

##### `exportToHtml(path: string): void`
导出会话为 HTML 文件。

**设计模式**：
- **外观模式（Facade）**：`AgentSession` 封装了复杂的子系统（agent、session、settings、models），提供简洁的接口
- **观察者模式**：通过 `subscribe()` 支持事件订阅
- **模板方法**：`prompt()` 定义了固定的执行步骤

**潜在问题**：
- 文件过大（1000+ 行），包含大量职责，违反单一职责原则
- 多个方法涉及复杂的异步操作，错误处理分散
- `subscribe` 中的自动持久化逻辑与事件监听耦合

---

## 5.3 coding-agent/src/core/sdk.ts — SDK 工厂

### 文件概述
`sdk.ts` 定义了 `createAgentSession()` 等 SDK 工厂函数，是外部应用嵌入 Pi 能力的主要入口。

### 核心函数

#### `createAgentSession(options?: CreateAgentSessionOptions): Promise<CreateAgentSessionResult>`
**功能**：创建 AgentSession 实例

**参数**：
- `cwd`：工作目录
- `agentDir`：配置目录
- `model`：模型
- `thinkingLevel`：思考级别
- `scopedModels`：模型循环列表
- `tools`：工具白名单
- `excludeTools`：工具黑名单
- `customTools`：自定义工具
- `resourceLoader`：资源加载器
- `sessionManager`：会话管理器
- `settingsManager`：设置管理器

**执行流程**：
1. 创建或获取 ModelRuntime
2. 创建或获取 SettingsManager
3. 创建或获取 ResourceLoader
4. 创建或获取 SessionManager
5. 加载扩展
6. 创建工具定义（内置 + 自定义）
7. 构建 system prompt
8. 创建 Agent 实例
9. 创建 AgentSession 实例
10. 返回结果

#### 工具工厂函数

##### `createBashTool(options?: BashToolOptions): AgentToolDefinition`
创建 bash 工具定义。

##### `createCodingTools(options?: ToolsOptions): AgentToolDefinition[]`
创建所有编码工具（read、write、edit、bash）。

##### `createReadOnlyTools(options?: ToolsOptions): AgentToolDefinition[]`
创建只读工具（read、find、grep、ls）。

##### `createEditTool(options?: EditToolOptions): AgentToolDefinition`
创建编辑工具。

##### `createReadTool(options?: ReadToolOptions): AgentToolDefinition`
创建读取工具。

##### `createWriteTool(options?: WriteToolOptions): AgentToolDefinition`
创建写入工具。

##### `createFindTool(options?: FindToolOptions): AgentToolDefinition`
创建查找工具。

##### `createGrepTool(options?: GrepToolOptions): AgentToolDefinition`
创建 grep 工具。

##### `createLsTool(options?: LsToolOptions): AgentToolDefinition`
创建 ls 工具。

**设计模式**：
- **工厂模式**：每个工具都是一个工厂函数，接收 options，返回工具定义
- **建造者模式**：`createAgentSession` 通过组合多个工厂函数构建完整的 session

---

## 5.4 coding-agent/src/core/session-manager.ts — 会话持久化

### 文件概述
`session-manager.ts` 定义了 `SessionManager` 类，负责会话的 CRUD 操作和 JSONL 持久化。

### 核心类

#### `SessionManager`
**职责**：会话的创建、打开、恢复、分叉、列出

**关键方法**：

##### `static create(cwd: string, sessionDir?: string, options?: NewSessionOptions): SessionManager`
创建新会话。
1. 生成 UUIDv7 作为 sessionId
2. 创建 sessionDir 目录（如果不存在）
3. 创建 `<sessionId>.jsonl` 文件
4. 写入 SessionHeader entry
5. 返回 SessionManager 实例

##### `static open(path: string, sessionDir?: string, cwd?: string): SessionManager`
打开已有会话。
1. 读取 JSONL 文件
2. 解析所有 entries
3. 重建 SessionContext
4. 返回 SessionManager 实例

##### `static continueRecent(cwd: string, sessionDir?: string): SessionManager`
继续最近使用的会话。
1. 列出当前项目的所有会话
2. 按最后修改时间排序
3. 打开最近的会话

##### `static forkFrom(sourcePath: string, cwd: string, sessionDir?: string, options?: NewSessionOptions): SessionManager`
分叉会话。
1. 复制源会话文件
2. 生成新的 sessionId
3. 返回新 SessionManager 实例

##### `static list(cwd: string, sessionDir?: string, onProgress?: (sessions: SessionInfo[]) => void): Promise<SessionInfo[]>`
列出当前项目的所有会话。

##### `static listAll(sessionDir?: string, onProgress?: (sessions: SessionInfo[]) => void): Promise<SessionInfo[]>`
列出所有项目的会话（全局搜索）。

##### `appendEntry(entry: SessionEntry): void`
追加 entry 到 JSONL 文件。
1. 序列化 entry 为 JSON
2. 追加到文件末尾
3. 更新内存中的 context

##### `buildSessionContext(): SessionContext`
构建当前会话上下文（用于恢复）。

**SessionEntry 类型体系**：
```typescript
type SessionEntry =
  | SessionHeaderEntry      // 会话头（版本、创建时间）
  | SessionInfoEntry        // 会话信息（名称等）
  | SessionMessageEntry     // 消息（user/assistant）
  | ToolResultEntry         // 工具结果
  | FileEntry               // 文件操作记录
  | CustomEntry             // 自定义消息
  | ModelChangeEntry        // 模型变更
  | ThinkingLevelChangeEntry // 思考级别变更
  | CompactionEntry         // 压缩记录
  | BranchSummaryEntry      // 分支摘要
  | NewSessionEntry         // 新会话标记
```

**设计模式**：
- **仓储模式（Repository）**：`SessionManager` 封装了会话的持久化细节，上层不需要知道 JSONL 格式
- **工作单元（Unit of Work）**：`appendEntry` 同时更新内存状态和持久化存储

**JSONL 格式**：
- 每行一个 JSON 对象
- Entry 包含 `type` 字段区分类型
- 追加写入，崩溃安全
- 支持版本迁移（`migrateSessionEntries`）

---

## 5.5 coding-agent/src/core/extensions/ — 扩展系统

### 文件概述
扩展系统位于 `core/extensions/` 目录，提供扩展的发现、加载、运行能力。

### 核心接口

#### `Extension`
扩展的主要接口：
```typescript
interface Extension {
  name: string;
  description?: string;
  commands?: ExtensionCommand[];
  tools?: RegisteredTool[];
  events?: ExtensionEvents;
  ui?: ExtensionUI;
}
```

#### `ExtensionFactory`
扩展工厂函数类型：
```typescript
type ExtensionFactory = (context: ExtensionContext) => Extension | Promise<Extension>;
```

#### `ExtensionContext`
扩展上下文，提供给 factory 函数：
```typescript
interface ExtensionContext {
  cwd: string;
  agentDir: string;
  logger: Logger;
  config: Config;
  // ...
}
```

### 核心类

#### `ExtensionRunner`
**职责**：扩展运行时分发器

**关键方法**：
- `emit(event: ExtensionEvent): Promise<void>`：分发事件到各扩展的钩子
- `getExtensions(): Extension[]`：获取所有加载的扩展
- `getTools(): RegisteredTool[]`：获取所有扩展的工具
- `getCommands(): RegisteredCommand[]`：获取所有扩展的命令
- `dispose(): Promise<void>`：清理扩展资源

### 核心函数

#### `discoverAndLoadExtensions(options): Promise<LoadExtensionsResult>`
发现并加载扩展。
1. 扫描内置扩展
2. 扫描 `.pi/extensions` 目录
3. 扫描 `--extensions` 参数指定的路径
4. 对每个扩展文件，使用 `jiti` 导入
5. 调用 factory 函数创建扩展实例
6. 收集加载结果（成功/失败）

#### `wrapRegisteredTool(tool: RegisteredTool): AgentToolDefinition`
将扩展注册的工具包装为 agent 可用的工具定义。

#### `wrapRegisteredTools(tools: RegisteredTool[]): AgentToolDefinition[]`
批量包装工具。

#### `defineTool(definition: ToolDefinition): RegisteredTool`
扩展用于注册工具的工具函数。

### 事件钩子类型

扩展可以订阅的事件钩子（30+ 种）：
- `beforeAgentStart`：agent 开始前
- `agentStart`：agent 开始
- `agentEnd`：agent 结束
- `turnStart`：轮次开始
- `turnEnd`：轮次结束
- `beforeToolCall`：工具调用前（可阻止）
- `afterToolCall`：工具调用后（可覆盖结果）
- `onMessage`：消息事件
- `onInput`：输入事件
- `beforeProviderRequest`：provider 请求前
- `beforeProviderHeaders`：provider headers 前
- `sessionBeforeCompact`：会话压缩前
- `sessionBeforeFork`：会话分叉前
- `sessionBeforeSwitch`：会话切换前
- `sessionShutdown`：会话关闭
- 更多...

**设计 Rationale**：
- **AOP（面向切面编程）**：事件钩子允许扩展在 agent 生命周期的任意点注入逻辑
- **开放-封闭原则**：核心稳定，扩展通过钩子添加能力
- **类型安全**：所有事件和钩子都有完整的 TypeScript 类型定义

---

## 5.6 coding-agent/src/core/tools/ — 内置工具

### 文件概述
`core/tools/` 目录包含 7 个内置工具的实现。每个工具是一个工厂函数，接收 options，返回 `AgentToolDefinition`。

### 通用结构

每个工具文件遵循相同的模式：
1. 定义输入参数 schema（使用 TypeBox）
2. 定义 details 类型（工具执行的详细信息）
3. 定义 operations 接口（便于测试和扩展）
4. 定义 `createXxxToolDefinition(options)` 工厂函数
5. 定义 `createLocalXxxOperations()` 创建默认 operations

### 各工具详解

#### bash.ts — Bash 工具
**功能**：执行 shell 命令

**输入参数**：
- `command`：要执行的命令
- `timeout`：超时时间（毫秒）
- `description`：命令描述

**核心逻辑**：
1. `createLocalBashOperations()` 创建操作集
2. `executeBashWithOperations()` 使用 `cross-spawn` 执行
3. 支持超时、输出截断、环境恢复
4. 通过 `onUpdate` 回调流式返回部分结果

**安全考虑**：
- 命令执行在用户的 shell 环境中
- 无内置沙箱（文档建议容器化）
- Project Trust 控制项目资源访问

#### edit.ts — 编辑工具
**功能**：精确编辑文件（字符串替换）

**输入参数**：
- `file_path`：文件路径
- `old_string`：要替换的字符串
- `new_string`：替换后的字符串
- `replace_all`：是否替换所有匹配

**核心逻辑**：
1. 读取文件内容
2. 查找 `old_string`
3. 检测多个匹配（如果 `replace_all` 为 false）
4. 执行替换
5. 写入文件
6. 生成 unified diff（通过 `edit-diff.ts`）

#### read.ts — 读取工具
**功能**：读取文件内容

**输入参数**：
- `file_path`：文件路径
- `offset`：起始行
- `limit`：行数限制

**核心逻辑**：
1. 读取文件内容
2. 支持行范围（offset/limit）
3. 支持图像文件（自动缩放/格式转换）
4. 输出截断（防止超出上下文窗口）

#### write.ts — 写入工具
**功能**：写入文件

**输入参数**：
- `file_path`：文件路径
- `content`：文件内容

**核心逻辑**：
1. 检查文件是否存在（不存在则创建）
2. 原子写入（通过文件锁）
3. 通过文件变更队列（`file-mutation-queue`）序列化写操作

#### find.ts — 查找工具
**功能**：按名称模式查找文件

**输入参数**：
- `pattern`：匹配模式
- `path`：搜索路径

**核心逻辑**：
1. 使用 `glob` 进行文件匹配
2. 遵守 `.gitignore` 规则
3. 返回匹配的文件列表

#### grep.ts — 搜索工具
**功能**：内容搜索

**输入参数**：
- `pattern`：正则表达式
- `path`：搜索路径
- `context`：上下文行数

**核心逻辑**：
1. 使用正则表达式搜索文件内容
2. 支持上下文行（前后 N 行）
3. 返回匹配结果

#### ls.ts — 列表工具
**功能**：列出目录内容

**输入参数**：
- `path`：目录路径

**核心逻辑**：
1. 读取目录内容
2. 返回文件元数据（名称、大小、类型）

### 工具定义包装

#### `tool-definition-wrapper.ts`
将工具定义转换为 agent 可用的 `AgentToolDefinition`。

#### `output-accumulator.ts`
工具输出累积器，用于流式收集工具输出。

#### `truncate.ts`
工具输出截断逻辑，防止超出上下文窗口。

#### `path-utils.ts`
路径处理工具函数。

---

## 5.7 coding-agent/src/core/compaction/ — 上下文压缩

### 文件概述
`core/compaction/` 目录包含上下文压缩的实现，用于在对话过长时压缩历史消息。

### 核心函数

#### `shouldCompact(context, model): boolean`
**功能**：判断是否需要压缩

**逻辑**：
1. `estimateContextTokens(context)` 估算当前 token 数
2. 与模型的上下文窗口阈值比较
3. 超过阈值返回 true

#### `findCutPoint(messages): CutPointResult`
**功能**：找到最佳压缩点

**逻辑**：
1. 从消息列表末尾开始扫描
2. 找到最后一个 `new_session` 或 `compaction` entry
3. 确保保留最近的完整轮次
4. 返回压缩点索引和保留的消息数

#### `generateSummary(conversation: string): Promise<string>`
**功能**：生成摘要

**逻辑**：
1. 将历史消息序列化为文本（`serializeConversation`）
2. 调用 LLM 生成结构化摘要
3. 返回摘要文本

#### `prepareCompaction(context): CompactionPreparation`
**功能**：准备压缩

**逻辑**：
1. 查找压缩点
2. 分离要压缩的消息和保留的消息
3. 序列化要压缩的消息

#### `compact(context, model, options): Promise<CompactionResult>`
**功能**：执行压缩

**逻辑**：
1. `prepareCompaction()` 准备
2. `generateSummary()` 生成摘要
3. 构建新的上下文（摘要 + 保留消息）
4. 返回压缩结果

#### `generateBranchSummary(entries): Promise<string>`
**功能**：生成分支摘要

**逻辑**：
1. 收集分支的所有 entries
2. 序列化为文本
3. 调用 LLM 生成摘要

### 辅助函数

#### `estimateTokens(messages): number`
估算消息列表的 token 数。

#### `estimateContextTokens(context): number`
估算整个上下文的 token 数。

#### `serializeConversation(messages): string`
将消息序列化为文本（用于摘要生成）。

#### `calculateContextTokens(context): number`
精确计算上下文的 token 数（如果模型支持）。

---

## 5.8 coding-agent/src/modes/interactive/ — 交互模式

### 文件概述
`modes/interactive/` 目录包含交互模式（TUI）的实现。

### 核心类

#### `InteractiveMode`
**位置**：`modes/interactive/interactive-mode.ts`
**职责**：全屏 TUI 交互模式

**关键方法**：

##### `init(): Promise<void>`
初始化 TUI。
1. 创建 TUI 引擎
2. 设置终端（raw mode、alternate screen）
3. 初始化组件
4. 设置 keybinding

##### `run(): Promise<void>`
运行交互模式主循环。
1. 显示欢迎消息
2. 进入事件循环
3. 处理用户输入
4. 渲染 agent 响应
5. 处理选择器交互

##### `stop(): void`
停止交互模式。
1. 恢复终端
2. 清理资源

### UI 组件

`modes/interactive/components/` 目录包含 30+ 个 UI 组件：

- **editor.ts**：多行文本编辑器
- **markdown.ts**：Markdown 渲染器
- **input.ts**：输入框
- **select-list.ts**：选择列表
- **settings-list.ts**：设置列表
- **box.ts**：边框容器
- **loader.ts**：加载指示器
- **diff.ts**：差异渲染器
- **footer.ts**：底部状态栏
- **model-selector.ts**：模型选择器
- **session-selector.ts**：会话选择器
- **theme-selector.ts**：主题选择器
- **thinking-selector.ts**：思考级别选择器
- **user-message.ts**：用户消息渲染器
- **assistant-message.ts**：助手消息渲染器
- **bash-execution.ts**：bash 执行渲染器
- **tool-execution.ts**：工具执行渲染器
- 更多...

### 主题系统

`modes/interactive/theme/` 目录包含主题系统：

- **theme.ts**：主题接口和加载
- **dark.json**：暗色主题
- **light.json**：亮色主题
- **theme-controller.ts**：主题控制器
- **theme-schema.json**：主题 schema

---

## 5.9 coding-agent/src/modes/rpc/ — RPC 模式

### 文件概述
`modes/rpc/` 目录包含 RPC 模式的实现。

### 核心类

#### `RpcMode`
**位置**：`modes/rpc/rpc-mode.ts`
**职责**：RPC 协议处理

**关键方法**：

##### `run(): Promise<void>`
运行 RPC 模式。
1. 监听 stdin 的 JSONL 输入
2. 解析命令
3. 执行命令
4. 输出响应和事件到 stdout

### 协议类型

#### `RpcCommand`
外部进程发送的命令类型：
- `initialize`：初始化连接
- `prompt`：发送用户消息
- `get_state`：获取状态
- `new_session`：创建新会话
- `switch_session`：切换会话
- `fork`：分叉会话
- `set_session_name`：设置会话名称
- 更多...

#### `RpcResponse`
Pi 返回的响应类型：
- `success`：是否成功
- `data`：响应数据
- `error`：错误信息

#### `RpcEvent`
Pi 发射的事件类型：
- agent_start/agent_end
- turn_start/turn_end
- message_start/message_update/message_end
- tool_execution_start/tool_execution_update/tool_execution_end
- 更多...

### JSONL 协议

- 每行一个 JSON 对象
- 命令和响应使用 `id` 字段关联
- 事件使用 `type: "event"` 标识
- UI 请求/响应使用 `ui_request`/`ui_response` 类型

---

## 5.10 agent/src/agent-loop.ts — 核心 Agent 循环

### 文件概述
`agent-loop.ts` 是 Pi 的核心，定义了 `agentLoop()` 和 `runAgentLoop()` 函数。

### 核心函数

#### `agentLoop(prompts, context, config, signal?, streamFn?): EventStream<AgentEvent, AgentMessage[]>`
**位置**：`agent-loop.ts:31`
**功能**：启动一个新的 agent loop

**流程**：
1. 创建 `EventStream`
2. 启动异步的 `runAgentLoop`
3. 返回事件流（调用者可以立即订阅）

#### `runAgentLoop(prompts, context, config, emit, signal?, streamFn?)`
**位置**：`agent-loop.ts:95`
**功能**：执行 agent loop 的核心逻辑

**详细流程**：
1. 发射 `agent_start`
2. 将 prompts 追加到 context.messages
3. **循环**：
   a. 发射 `turn_start`
   b. 准备上下文：
      - 调用 `transformContext`（可选）
      - 调用 `convertToLlm` 转换为 LLM 格式
   c. 调用 LLM：
      - 使用 `streamFn` 或默认的 `streamSimple`
      - 流式处理响应
      - 发射 `message_start`/`message_update`/`message_end`
   d. 将 assistant message 追加到 context
   e. 处理工具调用：
      - 如果没有工具调用，跳到 g
      - 如果有工具调用：
        - 对每个工具调用：
          - 发射 `tool_execution_start`
          - 调用 `beforeToolCall`
          - 如果未被阻止，执行工具
          - 调用 `afterToolCall`
          - 发射 `tool_execution_end`
          - 将 tool_result 追加到 context
   f. 发射 `turn_end`
   g. 检查停止条件：
      - `shouldStopAfterTurn` 返回 true → 停止
      - 有 steering 消息 → 注入并继续
      - 有 follow-up 消息 → 注入并继续
      - 没有更多 → 停止
   h. `prepareNextTurn` 可以更新 context/model/thinkingLevel
4. 发射 `agent_end`
5. 返回新消息

#### `agentLoopContinue(context, config, signal?, streamFn?): EventStream<AgentEvent, AgentMessage[]>`
**位置**：`agent-loop.ts:64`
**功能**：从当前上下文继续（无新消息）

**约束**：
- context.messages 不能为空
- 最后一条消息不能是 assistant（否则 LLM 会拒绝）

### 工具执行模式

#### 串行模式（`toolExecution: "sequential"`）
每个工具调用依次执行：prepare → execute → finalize → 下一个。

#### 并行模式（`toolExecution: "parallel"`）
1. 依次 preflight（prepare）所有工具调用
2. 并发执行所有被允许的工具
3. 按完成顺序发射 `tool_execution_end`
4. 按原始顺序发射 tool-result message artifacts

**默认**：并行模式（更高效）。

### 事件类型

```typescript
type AgentEvent =
  | { type: "agent_start" }
  | { type: "agent_end"; messages: AgentMessage[] }
  | { type: "turn_start" }
  | { type: "turn_end"; message: AgentMessage; toolResults: ToolResultMessage[] }
  | { type: "message_start"; message: AgentMessage }
  | { type: "message_update"; message: AgentMessage; assistantMessageEvent: AssistantMessageEvent }
  | { type: "message_end"; message: AgentMessage }
  | { type: "tool_execution_start"; toolCallId: string; toolName: string; args: any }
  | { type: "tool_execution_update"; toolCallId: string; toolName: string; args: any; partialResult: any }
  | { type: "tool_execution_end"; toolCallId: string; toolName: string; result: any; isError: boolean }
```

**设计 Rationale**：
- **事件驱动**：agent loop 通过事件流与外部通信，不直接依赖 UI
- **流式处理**：LLM 响应和工具执行都是流式的，UI 可以实时更新
- **可扩展**：通过 hooks（beforeToolCall、afterToolCall 等）和队列（steering、follow-up）提供扩展点
- **错误编码**：错误通过事件流的 stopReason 编码，不抛出异常（避免中断循环）

---

## 5.11 agent/src/agent.ts — Agent 类

### 文件概述
`agent.ts` 定义了 `Agent` 类，管理 agent 状态和事件。

### 核心类

#### `Agent`
**职责**：agent 状态管理、事件发射

**关键属性**：
- `state`：`AgentState`（systemPrompt、model、thinkingLevel、tools、messages）
- `context`：`AgentContext`（当前上下文）

**关键方法**：

##### `agentLoop(prompts, config): EventStream<AgentEvent, AgentMessage[]>`
启动 agent loop。

##### `agentLoopContinue(config): EventStream<AgentEvent, AgentMessage[]>`
从当前上下文继续。

##### `subscribe(listener): Unsubscribe`
订阅 agent 事件。

##### `abort(): void`
中止当前运行。

**设计模式**：
- **观察者模式**：通过 `subscribe()` 支持事件订阅
- **状态模式**：`AgentState` 封装了 agent 的所有状态

---

## 5.12 ai/src/models.ts — Models 类

### 文件概述
`models.ts` 定义了 `Models` 类，管理 Provider 集合和流式调用。

### 核心类

#### `Models`
**职责**：Provider 集合、模型发现、流式调用

**关键方法**：

##### `addProvider(provider: Models): void`
添加 provider 到集合。

##### `getProvider(id: string): Provider | undefined`
获取 provider。

##### `streamSimple(model, context, options): AssistantMessageEventStream`
**功能**：流式调用 LLM

**流程**：
1. 解析认证（`resolveProviderAuth`）
2. 获取 provider 实例
3. 调用 provider 的 stream 方法
4. 返回事件流

##### `refreshModels(options): Promise<ModelsRefreshResult>`
**功能**：刷新模型列表

**流程**：
1. 对每个 provider 调用 `refreshModels`
2. 合并结果
3. 返回错误映射

### Provider 接口

```typescript
interface Provider<TApi extends Api = Api> {
  readonly id: string;
  readonly name: string;
  readonly baseUrl?: string;
  readonly headers?: ProviderHeaders;
  auth: ProviderAuth;
  models: Model[];
  refreshModels(options): Promise<void>;
  stream(model, context, options): AssistantMessageEventStream;
}
```

---

## 5.13 orchestrator/src/supervisor.ts — 编排器

### 文件概述
`supervisor.ts` 定义了 `OrchestratorSupervisor` 类，管理多个 Pi 实例。

### 核心类

#### `OrchestratorSupervisor`
**职责**：多实例生命周期管理

**关键方法**：

##### `createInstance(options): InstanceRecord`
创建新实例。
1. 启动 `pi --rpc` 子进程
2. 创建 `InstanceRecord`
3. 保存到 `instances.json`
4. 返回记录

##### `destroyInstance(id: string): Promise<void>`
销毁实例。
1. 杀死子进程
2. 清理订阅者
3. 从 `instances.json` 移除

##### `sendCommand(id: string, command: RpcCommand): Promise<RpcResponse>`
发送命令到实例。
1. 找到实例
2. 写入命令到子进程 stdin
3. 读取响应
4. 更新实例状态

##### `subscribe(id: string, listener: AgentSessionEventListener): Unsubscribe`
订阅实例事件。
1. 找到实例
2. 添加订阅者
3. 返回取消订阅函数

**设计模式**：
- **中介者模式**：`Supervisor` 协调多个实例的交互
- **观察者模式**：通过 `subscribe()` 支持事件订阅

---

## 5.14 tui/src/tui.ts — TUI 引擎

### 文件概述
`tui.ts` 定义了 TUI 引擎，提供差分渲染和终端控制。

### 核心概念

#### 差分渲染
TUI 引擎使用差分渲染技术，只更新终端中发生变化的部分，而不是每次重绘整个屏幕。这大大提高了渲染效率。

**实现原理**：
1. 维护两个缓冲区：当前帧和上一帧
2. 比较两帧差异
3. 只输出差异部分到终端

#### 终端控制
- 进入/退出 raw mode
- 进入/退出 alternate screen
- 光标移动和隐藏
- 颜色和样式

---

☕️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../res/promotion.jpg)