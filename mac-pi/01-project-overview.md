# 第一部分：项目概述（Project Overview）

## 1.1 项目定位与目标

### Pi 是什么

Pi 是一个**自扩展的编码代理（coding agent）命令行工具**，运行在终端中，让开发者能够通过自然语言与大型语言模型（LLM）协作完成编码任务。它的核心交付物是一个名为 `pi` 的可执行命令（同时提供 Node.js 和 Bun 两种运行时形态），用户启动后进入一个交互式终端 TUI，向 LLM 发送提示词，LLM 则通过内置工具（读文件、写文件、编辑文件、执行 bash 命令、搜索文件、grep、ls）直接操作用户的代码仓库。

### 解决什么问题

传统的 AI 编码助手（IDE 插件、Web 聊天界面）存在以下痛点：

1. **上下文割裂**：用户需要手动复制代码到聊天窗口，再手动应用返回的修改。
2. **工具受限**：大多数助手只能生成代码，无法直接执行命令、运行测试、验证修改。
3. **provider 锁定**：用户被单一 LLM 提供商绑定，无法灵活切换。
4. **缺乏自扩展能力**：用户无法为特定工作流添加自定义工具或命令。

Pi 通过以下方式解决：

- **直接在终端工作目录操作**：agent 读写文件、执行 shell 命令都在用户的 Git 仓库中进行，无需手动搬运。
- **工具调用（tool calling）**：LLM 不只是生成文本，而是调用 `bash`、`edit`、`read`、`write`、`find`、`grep`、`ls` 等工具，形成"思考-执行-观察"循环。
- **43 个 LLM provider 统一接口**：通过 `@earendil-works/pi-ai` 包，用户可以在 OpenAI、Anthropic、Google、AWS Bedrock、Azure、xAI、Groq、Mistral、DeepSeek、小米、MiniMax 等 43 个提供商之间无缝切换，甚至通过 OpenRouter 接入更多模型。
- **自扩展架构（self-extensible）**：通过扩展系统（extension system），开发者可以用 TypeScript 编写自定义工具、命令、UI 组件、消息渲染器、事件钩子，无需修改 Pi 核心代码。

### 核心价值主张

- **终端原生**：为习惯终端的开发者设计，不依赖 GUI，支持 tmux/SSH 场景。
- **provider 无关**：统一的 LLM API 抽象层，模型切换只需 `--model` 参数。
- **可审计**：会话以 JSONL 格式持久化，每一步操作（用户消息、工具调用、工具结果、模型输出、压缩摘要）都有完整记录。
- **自扩展**：extension API 提供 30+ 种事件钩子，覆盖 agent 生命周期、工具执行、UI 渲染、输入处理等。
- **安全默认**：项目信任机制（project trust）、供应链加固（依赖锁定 + 审计 + 容器化指南）。

### 与竞品的差异

| 维度 | Pi | Claude Code | Aider | OpenAI Codex |
|------|-----|-------------|-------|--------------|
| 运行形态 | CLI + TUI + SDK + RPC | CLI + SDK | CLI | 云/CLI |
| Provider 数量 | 43+ | 1（Anthropic）+ Bedrock | 多 | 1（OpenAI） |
| 扩展系统 | 完整 extension API | 有限（hooks） | 无 | 无 |
| 会话格式 | JSONL（本地、可审计） | 专有 | 专有 | 云存储 |
| 开源程度 | MIT（完全开源） | 闭源 | Apache 2.0 | 部分 |
| 本地优先 | 完全本地运行 | 本地+云 | 本地 | 云端为主 |
| 差分渲染 TUI | 自研 | 内置 | 无 | 无 |
| 多实例编排 | orchestrator 包 | 无 | 无 | 无 |

---

## 1.2 目标用户画像

### 主要用户

1. **专业软件工程师**：日常在终端工作，希望 AI 助手直接操作代码仓库，而不是在 IDE 和聊天窗口之间切换。典型场景：修复 bug → agent 读取相关文件 → 执行测试 → 修改代码 → 验证通过。

2. **AI 工具研究者/高级用户**：需要对比不同 LLM 在相同编码任务上的表现，需要灵活切换 provider 和模型，需要自定义工具扩展 agent 能力。

3. **需要本地+隐私控制的用户**：敏感代码不能上传到云端，Pi 支持完全本地运行（通过 Cerebras、Groq 等，或自部署模型通过 OpenAI-compatible API），所有会话数据存储在本地 `.pi` 目录。

4. **工具/平台开发者**：通过 SDK 将 Pi 的 agent 能力嵌入自己的应用（IDE 插件、CI 流水线、自动化平台），通过 RPC 模式与 Pi 进程通信。

### 典型使用场景

- **交互式开发**：`pi` 进入 TUI，对话式编码，agent 使用 bash 工具运行测试、使用 edit 工具修改代码。
- **非交互批处理**：`echo "fix the tests" | pi` 或 `pi -p "generate docs"`，管道输入，输出到 stdout。
- **SDK 集成**：`createAgentSession()` 在自己的 Node.js 应用中嵌入 agent 能力。
- **RPC 集成**：IDE 插件通过 JSONL 协议与 Pi 进程通信，获取完整的 agent 事件流和 UI 能力。
- **多实例编排**：orchestrator 同时管理多个 Pi 实例，用于并行任务或团队协作。

---

## 1.3 技术栈完整清单

### 语言与运行时

| 技术 | 版本 | 用途 |
|------|------|------|
| TypeScript | 5.9.3 | 主要开发语言（erasable syntax / strip-only mode） |
| Node.js | ≥22.19.0 | 运行时（要求原生 strip-only TS 支持） |
| Bun | 最新稳定 | 替代运行时 + 单二进制编译（`bun build --compile`） |
| C (Native) | - | macOS `darwin-modifiers.c`、Windows `win32-console-mode.c` 原生终端控制 |
| Rust → WASM | - | `@silvia-odwyer/photon-node` 图像处理库（photon_rs_bg.wasm） |

### 构建与工具链

| 技术 | 版本 | 用途 |
|------|------|------|
| @typescript/native-preview (tsgo) | 7.0.0-dev.20260120.1 | TypeScript 原生编译器（Node strip-only 模式），替代 tsc |
| esbuild | 0.28.1 | 脚本打包（scripts/ 目录下的构建/发布脚本） |
| Biome | 2.3.5 | 代码检查 + 格式化（替代 ESLint + Prettier） |
| husky | 9.1.7 | Git hooks（pre-commit 检查 lockfile） |
| jiti | 2.7.0 | 运行时 TypeScript 执行（加载扩展、配置） |
| shx | 0.4.0 | Shell 命令跨平台（构建脚本中 cp/rm） |
| tsx | 4.22.1 | 开发时 TS 执行 |

### 核心依赖

| 包 | 版本 | 用途 |
|------|------|------|
| typebox | 1.1.38 | 类型安全的 schema 定义与验证（工具参数 schema） |
| undici | 8.5.0 | HTTP 客户端（provider API 请求、代理支持） |
| chalk | 5.6.2 | 终端着色 |
| cross-spawn | 7.0.6 | 跨平台子进程（bash 工具执行） |
| diff | 8.0.4 | 文本差异生成（edit 工具 unified diff） |
| glob | 13.0.6 | 文件模式匹配 |
| highlight.js | 10.7.3 | 代码语法高亮（HTML 导出、TUI 渲染） |
| yaml | 2.9.0 | YAML 解析（配置、frontmatter） |
| semver | 7.8.0 | 版本号解析与比较 |
| minimatch | 10.2.5 | 文件名模式匹配（.gitignore 风格） |
| ignore | 7.0.5 | .gitignore 规则解析 |
| proper-lockfile | 4.1.2 | 文件锁（会话并发保护） |
| hosted-git-info | 9.0.3 | Git 主机 URL 解析（GitHub/GitLab 等） |
| @mariozechner/clipboard | 0.3.9 (optional) | 系统剪贴板访问 |
| @silvia-odwyer/photon-node | 0.3.4 | Rust WASM 图像处理（缩放、格式转换） |

### 测试

| 工具 | 版本 | 用途 |
|------|------|------|
| vitest | 4.1.9 | 单元/集成测试框架 |

### CI/CD 与基础设施

| 工具 | 用途 |
|------|------|
| GitHub Actions | CI（check+test）、构建二进制、npm 发布、issue 管理 |
| npm trusted publishing | 通过 GitHub Actions OIDC 发布到 npm，无需本地 npm publish |
| npm-shrinklock.json | 发布包的传递依赖锁定 |

---

## 1.4 架构风格与理由

### 分层架构（Layered Architecture）

Pi 采用清晰的**三层模块分离**：

```
┌─────────────────────────────────────────────┐
│   @earendil-works/pi-coding-agent           │  ← 应用层
│   （CLI、会话管理、工具、扩展、TUI、RPC）     │
├─────────────────────────────────────────────┤
│   @earendil-works/pi-agent-core             │  ← Agent 运行时层
│   （agent loop、状态管理、harness、session）  │
├─────────────────────────────────────────────┤
│   @earendil-works/pi-ai                     │  ← LLM API 层
│   （统一 provider 接口、认证、模型发现）       │
└─────────────────────────────────────────────┘
```

**设计理由**：
- **关注点分离**：LLM 通信（ai）、agent 循环逻辑（agent-core）、应用/工具/UI（coding-agent）各自独立演进。
- **可独立复用**：`pi-ai` 可用于任何需要多 provider LLM API 的场景；`pi-agent-core` 可用于非编码类 agent。
- **可测试性**：每层可独立测试，`pi-agent-core` 有独立的 harness 测试配置。

### 事件驱动（Event-Driven）

agent loop 通过事件流（`EventStream<AgentEvent, AgentMessage[]>`）与外部通信。每个 agent 运行产生一系列事件：`agent_start` → `turn_start` → `message_start`/`message_update`/`message_end` → `tool_execution_start`/`update`/`end` → `turn_end` → `agent_end`。

**设计理由**：
- **流式 UI 更新**：TUI 可以实时渲染 LLM 的流式输出和工具执行进度。
- **解耦**：agent loop 不直接依赖 UI，UI 通过事件订阅更新。
- **扩展性**：扩展系统通过事件钩子注入自定义逻辑。

### 插件/扩展架构（Plugin/Extension Architecture）

Pi 提供完整的 extension API，扩展可以：
- 注册自定义工具（`defineTool`）
- 注册自定义 slash 命令
- 注册消息渲染器、UI 组件
- 订阅 30+ 种事件钩子（`beforeToolCall`、`afterToolCall`、`beforeAgentStart` 等）
- 注册自定义 provider（通过 extension）

**设计理由**：
- **开放-封闭原则**：核心功能稳定，新能力通过扩展添加。
- **社区生态**：第三方可以发布扩展包（如 `with-deps`、`custom-provider-anthropic`、`gondolin` 沙箱、`sandbox` 等示例）。
- **领域隔离**：特定领域能力（如 Gondolin 沙箱集成）不污染核心。

### JSONL 会话持久化

每个会话存储为 `.jsonl` 文件（每行一个 JSON 对象），记录完整的对话历史和元数据。

**设计理由**：
- **追加写入友好**：新消息追加到文件末尾，无需重写整个文件。
- **可审计**：每一步操作都有记录，可回放、可调试。
- **可恢复**：崩溃后可从上次写入位置恢复。
- **可分叉**：通过 `forkFrom` 复制会话文件，创建分支会话。

### Monorepo + npm Workspaces

**设计理由**：
- **原子提交**：跨包变更可以在一次提交中完成。
- **共享工具链**：Biome、TypeScript 配置在根目录统一管理。
- **lockstep 版本**：所有包共享同一版本号，简化发布。

---

## 1.5 关键功能特性

### 运行模式

1. **Interactive Mode（交互模式）**：全屏 TUI，支持消息流式渲染、选择器（模型选择、会话选择、设置选择）、主题切换、keybinding。
2. **Print Mode（打印模式）**：非交互，输出纯文本到 stdout，支持管道输入。
3. **JSON Mode**：与 print 模式类似，但输出结构化 JSON。
4. **RPC Mode**：通过 stdin/stdout JSONL 协议与外部进程通信，暴露完整的 agent 能力和事件流。

### 工具系统

| 工具 | 功能 | 关键特性 |
|------|------|---------|
| `bash` | 执行 shell 命令 | 超时控制、输出截断、环境恢复、Windows 兼容 |
| `read` | 读取文件内容 | 行范围、字节限制、图像文件（含缩放/格式转换） |
| `write` | 写入文件 | 原子写入、文件变更队列 |
| `edit` | 精确编辑（字符串替换） | unified diff 生成、多匹配检测 |
| `find` | 按名称模式查找文件 | glob、.gitignore 感知 |
| `grep` | 内容搜索 | regex、上下文行、文件类型过滤 |
| `ls` | 列出目录内容 | 文件元数据、人类可读大小 |

### 会话管理

- **创建**：`SessionManager.create(cwd)` 创建新会话
- **恢复**：`SessionManager.open(path)` / `continueRecent(cwd)` 恢复已有会话
- **分叉**：`SessionManager.forkFrom(source)` 复制会话创建分支
- **分支树**：支持会话树（tree）操作，可视化会话分支
- **搜索**：按 ID 前缀搜索、跨项目全局搜索

### 上下文压缩（Compaction）

当对话超过模型的上下文窗口时，自动或手动触发压缩：
- **Token 估算**：`estimateTokens` / `estimateContextTokens`
- **Cut point 查找**：`findCutPoint` 找到最佳压缩点
- **摘要生成**：`generateSummary` 用 LLM 生成历史对话摘要
- **分支摘要**：`generateBranchSummary` 为会话分支生成摘要

### 扩展系统

- 扩展发现：从 `.pi/extensions` 目录、`--extensions` 参数、内置扩展加载
- 扩展 API：`Extension` 接口提供 `name`、`description`、`commands`、`tools`、`events`、`ui` 等
- 事件钩子：`beforeAgentStart`、`afterToolCall`、`onMessage` 等 30+ 种
- UI 组件：扩展可以注册自定义 React 风格组件（通过 TUI 渲染）

### Skills

- Markdown 文件定义，frontmatter 描述元数据
- 从 `.pi/skills` 目录加载
- 在 system prompt 中格式化后注入

### Prompt Templates

- 可复用的 prompt 模板文件
- 支持变量扩展（`expandPromptTemplate`）
- 从 `.pi/prompt-templates` 目录加载

### 主题系统

- 暗色/亮色主题（JSON 定义）
- 自定义主题（`--themes` 参数）
- 系统主题跟随（macOS）

### 认证系统

- API Key（环境变量 / `~/.pi/auth.json`）
- OAuth 流程（Anthropic、GitHub Copilot、OpenAI Codex、xAI）
- Device Code 流程（GitHub Copilot）
- PKCE 安全码交换
- 凭证加密存储

### 安全机制

- **Project Trust**：首次访问项目资源时提示用户确认
- **Supply-chain hardening**：依赖锁定、审计、lifecycle 脚本白名单
- **容器化支持**：Gondolin micro-VM、Docker、OpenShell 三种沙箱模式

---

## 1.6 非功能性需求

### 性能

- **启动速度**：通过 `PI_STARTUP_BENCHMARK` 环境变量测量启动时间；`timings.ts` 提供细粒度计时
- **流式渲染**：TUI 差分渲染，只更新变化的终端区域
- **工具执行**：支持并行工具执行（`toolExecution: "parallel"`），减少等待时间
- **二进制分发**：Bun 编译单二进制，启动速度优于 Node.js

### 扩展性

- **水平扩展**：orchestrator 包支持多实例管理
- **垂直扩展**：扩展系统允许无限添加自定义能力
- **Provider 扩展**：通过 extension 注册自定义 provider

### 安全性

- **Project Trust**：细粒度的项目资源访问控制
- **凭证安全**：加密存储、OAuth 短期令牌
- **供应链安全**：依赖锁定 + 审计 + 白名单
- **沙箱隔离**：三种容器化/沙箱模式
- **无内置权限系统**：明确文档说明默认以用户权限运行，需要时容器化

### 可用性

- **跨平台**：Windows、macOS、Linux、Termux（Android）
- **终端兼容**：支持 tmux、各种终端模拟器
- **可访问性**：keybinding 可配置、主题可切换

### 可维护性

- **代码质量**：Biome 严格检查、无 `any` 政策、erasable TypeScript
- **测试覆盖**：319 个测试文件，含单元/集成/回归测试
- **文档**：29 篇 docs 文档、AGENTS.md 开发规则、CONTRIBUTING.md 贡献指南
- **版本管理**：lockstep 版本、CHANGELOG 规范
- **Git 工作流**：多 agent 并发工作的 Git 安全规则

---

☕️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../res/promotion.jpg)