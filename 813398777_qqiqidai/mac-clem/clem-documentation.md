# clem — Continuously Looping Engineering Machines

> **安全、自托管的 Claude Code 智能体舰队运行框架** — 每个智能体通过内核级出口防火墙或零秘密凭证代理进行隔离。

*您基础设施上的 "docker-compose for Claude Code"。*

[![MIT License](https://img.shields.io/badge/license-Mit-green)](https://github.com/jahwag/clem/blob/main/LICENSE) [![Go Version](https://img.shields.io/badge/go-1.22+-00ADD8?logo=go)](https://github.com/jahwag/clem) [![Discord](https://img.shields.io/badge/Discord-community-5865F2?logo=discord)](https://discord.gg/pR4qeMH4u4)

**官方链接：** [github.com/jahwag/clem](https://github.com/jahwag/clem) · [myclementine.ai](https://myclementine.ai) · [Discord 社区](https://discord.gg/pR4qeMH4u4)

---

## 目录

1. [项目概述](#1-项目概述)
2. [核心特性](#2-核心特性)
3. [工作原理](#3-工作原理)
4. [安全模型](#4-安全模型)
5. [系统要求](#5-系统要求)
6. [安装指南](#6-安装指南)
7. [快速开始 (Quickstart)](#7-快速开始-quickstart)
8. [协调后端](#8-协调后端)
9. [Discord 配置](#9-discord-配置)
10. [GitHub 协调](#10-github-协调)
11. [GitHub 凭证](#11-github-凭证)
12. [CLI 参考](#12-cli-参考)
13. [clem.yaml 配置参考](#13-clemyaml-配置参考)
14. [密钥管理](#14-密钥管理)
15. [VPS 部署](#15-vps-部署)
16. [故障排除](#16-故障排除)
17. [社区与许可](#17-社区与许可)

---

## 1. 项目概述

**clem** 在任何 Linux 主机上 24/7 运行一组 Claude Code 智能体。每个智能体是一个独立的 OS 用户，运行在 systemd 下的 tmux 会话中。智能体通过 **Discord、Slack 或 GitHub Issues** 进行协调，接收任务、编写代码并提交 PR。看门狗自动重启崩溃的实例。您只需编写一个 `clem.yaml`，clem 负责配置 OS 用户并保持运行。

### 与传统方案的区别

clem 的独特之处在于：**选定的秘密和出口流量可以在 OS 层进行隔离，而非依赖智能体的配合。** 零秘密代理为智能体提供 HTTP 凭证的占位符，而另一个独立用户在出站请求时注入真实值。可选的出口隔离构建在同一代理之上：基于 UID 的内核防火墙强制所有流量通过 agent-vault 的 TLS-MITM 代理，该代理仅允许白名单中的主机，拒绝其余所有请求（非 root 智能体无法禁用其不拥有的防火墙）。

---

## 2. 核心特性

| 特性 | 说明 |
|------|------|
| **每智能体 OS 身份** | 每个智能体是独立的 Linux 用户 — 独立的家目录、git 身份、GitHub PR、Discord/Slack 机器人。崩溃边界是真实的。 |
| **内核出口隔离** | 基于 nftables 的每智能体 UID 防火墙强制所有流量通过本地回环代理；非 root 智能体无法禁用其不拥有的防火墙。无进程内逃生开关。可选的 `egress:` 块。 |
| **零秘密代理** | 代理智能体持有占位符 + 限定范围的仅注入令牌；真实凭证存储在*不同*用户拥有的保险库中，在出口时注入。`cat ~/.env` 对代理的密钥不产生任何有用信息。 |
| **多后端协调** | Discord、Slack 或 GitHub Issues，通过 `clem.yaml` 中可切换的 `coordination.backend:` 配置。一个配置旋钮。 |
| **多运行时** | `runtime: claude-code \| opencode \| codex`。在一个团队定义后混合 Anthropic 云、ChatGPT OAuth、Bedrock、Vertex、Ollama 和 OpenAI 兼容提供者。 |
| **加密密钥** | 每智能体 `.env` 在配置时从 age/sops 保险库生成。配置后永不离开主机。 |
| **自愈能力** | systemd + tmux 每智能体。看门狗定时器重启死亡或停滞的会话。仅在反复失败后触发告警。 |
| **自带模型** | 默认 Claude；一键切换到 Ollama Cloud/Bedrock/Vertex/本地模型。在本地 Gemma 4（通过 Ollama 的 E4B QAT）上进行了端到端测试。 |
| **实时运维** | `clem status` 显示每智能体健康状态。可选的每智能体 ttyd Web 终端 — 在浏览器中查看。 |
| **本地运行** | 笔记本、家庭服务器、树莓派、小型 VPS。无需 Kubernetes。无需云服务。 |

---

## 3. 工作原理

```
┌──────────────────────────────────────────────────────┐
│  Linux 主机  (笔记本 · 家庭服务器 · VPS · …)          │
│                                                      │
│  ┌──────────────┐   ┌──────────────┐                 │
│  │ OS 用户:     │   │ OS 用户:     │   systemd +     │
│  │ myteam-lead  │   │ myteam-worker│   tmux 每用户    │
│  │  claude 循环 │   │  claude 循环 │                 │
│  └──────┬───────┘   └──────┬───────┘                 │
│         └──── MCP (stdio) ─┘                         │
│                     │                                │
│  ┌──────────────────┴──────────────────┐             │
│  │  看门狗定时器 (每 5 分钟)            │             │
│  │  重启死亡智能体 → #alerts            │             │
│  └─────────────────────────────────────┘             │
└───────────────────┬──────────────────────────────────┘
                    │ 协调后端
          ┌─────────▼──────────────────────────┐
          │  Discord · Slack · GitHub Issues   │
          │  #tasks / 线程 · 标签 · gh          │
          └────────────────────────────────────┘
```

每个智能体运行一个循环：启动 `claude`、`opencode` 或 `codex`，注入提示，等待会话完成（最长 2 小时硬上限），休眠配置的 `iteration` 时长，重复。密钥加密存储在 `secrets.sops.yaml`（age/sops）中；`clem provision` 将其解密到主机的每智能体 `.env` 文件中。

---

## 4. 安全模型

自主智能体是不可信工作负载：提示注入、中毒依赖或模型错误可将其转变为外泄引擎。clem 的立场是**在 OS 层隔离它，而非礼貌地请求智能体配合。** 智能体将持有的每个凭证恰好获得四种处置之一：

| 处置方式 | 机制 | 真实秘密存储位置 | 防御的威胁 |
|----------|------|------------------|------------|
| **broker（代理）** | 凭证代理（独立 UID）将真实值注入智能体自己的出站 HTTPS | 代理内部 | API 密钥/Bearer 外泄 — 智能体仅持有占位符 |
| **sidecar（边车）** | 持有秘密的 MCP 服务器作为*不同*用户运行；智能体通过本地回环调用它并获得结果，从不接触密钥 | 边车内部 | 非 HTTP 凭证（网关令牌、内部数据库）和限定范围/只读访问 |
| **remove（移除）** | 删除凭证/MCP | 无 | 未使用的攻击面 |
| **egress firewall（出口防火墙）** | 每智能体 nftables UID 规则强制所有流量通过 agent-vault 的本地回环 TLS-MITM，仅允许白名单中的主机，内核拒绝其余所有流量（需要 `vault_broker`） | n/a | 向未批准主机外泄数据 |

### 为什么这比进程内或单容器沙箱更强

凭证代理将配置的 HTTP 秘密保留在**智能体无法读取的不同 OS 用户**下。出口隔离可在其上叠加**非 root 智能体无法禁用的基于 OS UID 的内核防火墙**。两个边界都不依赖进程内开关。仅代理的智能体仍具有无限制的网络访问；出口隔离的智能体也被代理，只能到达批准的主机。

> **诚实说明：** 凭证代理和 TLS-MITM 出口代理都是一个经过实战检验的 OSS 原语（[Infisical agent-vault](https://github.com/Infisical/agent-vault)）；内核边界是 nftables。clem 的贡献是 **OS 层组合** — 每智能体 UID 身份 + 隔离的秘密供应，配合可选的智能体无法绕过的内核路由。

两层都是**可选且默认关闭的**；现有舰队在您启用 `vault.backend: agent-vault`（代理）及其上的 `egress:` 块（隔离需要 `vault_broker`）之前不受影响。

- 完整威胁模型、保证和已知限制：[docs/threat-model.md](https://github.com/jahwag/clem/blob/main/docs/threat-model.md)
- 工作参考配置：[samples/secure-fleet/](https://github.com/jahwag/clem/blob/main/samples/secure-fleet/)

---

## 5. 系统要求

### 主机
任何带有 systemd 的 Linux 机器（推荐 Ubuntu 24.04）。可以是笔记本、家庭服务器、树莓派或云 VPS。必须安装 `tmux`、`git`、`python3`、`age`、`sops`、`yq` 和 `curl`。聊天后端还需要其 MCP 服务器在 `$PATH` 上（Discord 用 `mcp-discord`，Slack 用 `slack-mcp-server`）。GitHub 后端使用 `gh` CLI 代替协调 MCP。`clem provision` 安装每智能体运行时 CLI（Claude Code 或 opencode）。

### 本地机器
运行 `clem` 命令的地方（可以与主机相同）：
- `go` 1.22+（构建 `clem`）
- `age`、`sops`、`yq` — 本地编辑密钥
- `gh` — GitHub CLI

### 账户
一个协调后端（选一个）：
- **Discord** — 私有服务器 + 每智能体一个机器人令牌，或
- **Slack** — 工作区 + 每智能体一个 Slack 应用（机器人用户令牌 `xoxb-…`），或
- **GitHub Issues** — 带 `clem:*` 标签的任务板仓库 + 每智能体 `GH_TOKEN`（PR 使用相同令牌）

---

## 6. 安装指南

### 下载预编译二进制（Linux）

```bash
# x86-64
sudo curl -fsSL https://github.com/jahwag/clem/releases/latest/download/clem_linux_amd64 -o /usr/local/bin/clem && sudo chmod +x /usr/local/bin/clem

# arm64: 将 clem_linux_amd64 替换为 clem_linux_arm64
clem --version
```

### 从源码构建

```bash
git clone https://github.com/jahwag/clem.git
cd clem
go build -ldflags "-X github.com/jahwag/clem/cmd.Version=$(git describe --tags --always)" -o clem .
sudo install -m 0755 clem /usr/local/bin/clem
clem --version
```

### 升级

```bash
sudo clem update
```

---

## 7. 快速开始 (Quickstart)

完整的单机本地设置。如果要在单独的远程主机上配置，请参见 [VPS 部署](#15-vps-部署)。

### 无风险试用

沙盒化示例在 [`samples/`](https://github.com/jahwag/clem/blob/main/samples/README.md) 下：
- [`ollama-nemotron-4b`](https://github.com/jahwag/clem/blob/main/samples/ollama-nemotron-4b/README.md) — Discord + 本地 NVIDIA Nemotron 3 Nano 4B（~2.8 GB）
- [`slack-nemotron-4b`](https://github.com/jahwag/clem/blob/main/samples/slack-nemotron-4b/README.md) — Slack + 相同本地模型
- [`github-tasks`](https://github.com/jahwag/clem/blob/main/samples/github-tasks/README.md) — 通过 `gh` CLI 的 GitHub Issues 协调（无聊天 MCP）

### 初始化步骤

```bash
# 1. 新建团队仓库（替换为您的组织）
gh repo create my-team --private --clone && cd my-team

# 2. 生成配置脚手架（discord 是默认值；使用 --backend github 用于 GitHub Issues）
clem init
# clem init --backend github
```

编辑 `clem.yaml`：
- 设置 `project:`（成为 OS 用户前缀，例如 `myteam-lead`）
- 选择 `coordination.backend:`（`discord`、`slack` 或 `github`）
- **Discord/Slack：** 粘贴服务器/工作区 ID 和频道 ID — 参见 [Discord 配置](#9-discord-配置)
- **GitHub：** 设置 `github_repo` 和频道映射 — 参见 [GitHub 协调](#10-github-协调)
- 调整智能体 `name`、`role`、`model`、`iteration`（Go 时长：`30s`、`1m30s`、`2h`）、`runtime`、`provider`

编辑 `CLAUDE.shared.md` — 描述您的项目，填写 T2-T4 层级。编辑每个 `CLAUDE.<agentkey>.md` 添加每智能体详情。

```bash
# 3. 生成 age 密钥对 + .sops.yaml
clem vault init
```

```bash
# 4. 存储每智能体密钥（见下方 Discord/GitHub 配置）
clem vault set github        GH_TOKEN="ghp_..."
clem vault set discord-lead  DISCORD_TOKEN="Bot <lead-bot-token>"
clem vault set discord-worker DISCORD_TOKEN="Bot <worker-bot-token>"
```

```bash
# 5. 提交配置（secrets.sops.yaml 已加密 — 安全）
git add clem.yaml CLAUDE.*.md .sops.yaml secrets.sops.yaml
git commit -m "init team config"
git push
```

```bash
# 6. 配置 — 创建 OS 用户、安装服务、写入 .env
sudo clem provision
```

```bash
# 7. 认证每个智能体（每智能体打开浏览器）
sudo clem login
```

```bash
# 8. 启动并检查
sudo clem start
clem status
```

`clem status` 显示 systemd 状态、tmux 活跃度、令牌到期时间和每智能体最后日志行。一旦 `SYSTEMD=active` 和 `TMUX=yes`，智能体正在运行。

查看智能体日志：

```bash
clem logs lead
```

---

## 8. 协调后端

| 后端 | 任务板 | 认领方式 | 告警 | 唤醒机制 | 协调 MCP |
|------|--------|----------|------|----------|----------|
| `discord`（默认） | `#tasks` 论坛线程 | 线程前缀 `[TODO]` → `[IN PROGRESS]` | `#alerts` 频道 | `mcp-discord` 网关观察器 | `mcp-discord` |
| `slack` | `#tasks` 顶级消息 + 线程 | 对顶级消息的表情回应 | `#alerts` 频道 | 智能体每次迭代轮询 | `slack-mcp-server` |
| `github` | 带 `clem:*` 标签的 Issues | 通过 `gh issue edit` 自分配 | 在告警 Issue 上评论 | `clem-github-watch` 边车轮询 Issues API | 无（`gh` CLI） |

GitHub 协调在任务和 PR 之间关闭循环：工作存在于专用仓库的 Issues 中，输出落入带有 `Closes #N` 的 PR 中。聊天后端更适合实时操作员对话；GitHub 更适合您的真相源已在 GitHub 上的场景。

---

## 9. Discord 配置

创建**私有** Discord 服务器（非公开）。Discord 成员资格是访问控制层 — 智能体对任何能在频道中发帖的人执行指令。

### 需要创建的频道

| 名称 | 类型 | 用途 |
|------|------|------|
| `#general` | 文本 | 状态更新、操作员通信 |
| `#tasks` | 论坛 | 任务板 — 智能体在此认领线程 |
| `#alerts` | 文本 | 关键问题、看门狗告警 |
| `#lessons` | 论坛 | 事后分析、经验教训 |

启用**开发者模式**（设置 → 高级），然后右键点击服务器图标和每个频道，将其 ID 复制到 `clem.yaml` 中。

### 每智能体一个机器人

每个智能体一个应用程序，在任务线程中赋予每个智能体不同的名称和头像：

1. 访问 https://discord.com/developers/applications → **新建应用程序**（以智能体命名）
2. **Bot** 标签 → **重置令牌** → 复制
3. 启用 **Server Members Intent** 和 **Message Content Intent**
4. **OAuth2 → URL 生成器**：scopes 选 `bot`；permissions 选 `Send Messages`、`Read Message History`、`Attach Files`、`Manage Threads`、`Create Public Threads`
5. 在浏览器中打开生成的 URL，将机器人添加到服务器
6. 保存令牌：`clem vault set discord-<agentkey> DISCORD_TOKEN="Bot <token>"`

每个智能体重复以上步骤。

---

## 10. GitHub 协调

使用 GitHub Issues 作为任务板，替代 Discord 或 Slack。智能体使用 `gh` CLI 发现和认领工作；`clem provision` 安装每智能体的**Issue 观察器边车**，轮询 `api.github.com` 并在新的可认领 Issue 出现时唤醒 tmux 会话。

### 前置条件

1. 一个任务板仓库（可以与智能体编辑的代码仓库分开）
2. 该仓库上的标签：`clem:todo`、`clem:in-progress`、`clem:done`、`clem:blocked`
3. 两个用于看门狗告警和事后分析元 Issue；记录其 Issue 编号
4. 主机上的 `gh` CLI 和每个智能体保险库中的 `GH_TOKEN`

### 脚手架

```bash
clem init --backend github
```

### `clem.yaml` 结构

```yaml
coordination:
  backend: github
  github_repo: "your-org/your-tasks"
  channels:
    tasks:   "clem:todo"    # 标记可认领工作的标签
    alerts:  "12"           # 看门狗/严重告警的 Issue 编号
    lessons: "34"           # 事后分析的 Issue 编号

operator:
  github_logins: ["your-github-login"]

agents:
  lead:
    vaults: [github]
    # ...
```

### 任务板约定

| 概念 | GitHub 原语 |
|------|-------------|
| 任务 | 带 `clem:todo` 标签的开放 Issue |
| 状态 | 标签：`clem:todo` → `clem:in-progress` → `clem:done` 或 `clem:blocked` |
| 认领 | `gh issue edit N --add-assignee @me`，然后重新读取 Issue 确认赢得认领 |
| 更新 | 在 Issue 上评论 |
| 输出 | PR 正文包含 `Closes #N` |
| 告警 | 在告警 Issue 上评论（`channels.alerts`） |

### 配置的服务（GitHub 后端）

- `clem-<project>-<agent>.service` — 智能体运行器（不变）
- `clem-github-watch-<project>-<agent>.service` — 每 60 秒轮询未分配的 `clem:todo` Issue 并发送 `tmux send-keys` 唤醒智能体

启用 `egress:` 后，当 `backend: github` 时，`api.github.com` 会自动添加到出口白名单。观察器遵守与智能体相同的本地回环代理。

完整演练：[`samples/github-tasks/README.md`](https://github.com/jahwag/clem/blob/main/samples/github-tasks/README.md)

---

## 11. GitHub 凭证

每个智能体需要自己的 GitHub 令牌，以便 PR 和提交显示不同的作者。所有后端都需要；使用 `backend: github` 时，相同的令牌也驱动协调。

### 细粒度 PAT（最简单，适合个人项目）

1. 访问 https://github.com/settings/tokens?type=beta → **生成新令牌**
2. 选择目标仓库
3. 权限：`Contents`（读写）、`Pull requests`（读写）、`Issues`（读写）、`Workflows`（读写）
4. `clem vault set github GH_TOKEN="ghp_..."`（或每智能体保险库，如果想要单独的令牌）

### 每智能体 Git 身份

让 PR 以智能体的名称而非 root 作者。在 `clem provision` 后运行：

```bash
sudo -u myteam-lead git config --global user.name  "Amara"
sudo -u myteam-lead git config --global user.email "amara@yourproject.com"
sudo -u myteam-lead git config --global credential.helper store
echo "https://amara:ghp_...@github.com" | \
  sudo -u myteam-lead tee /home/myteam-lead/.git-credentials
```

每个智能体重复。

### GitHub App（推荐用于团队）

为每个智能体创建一个应用，每次迭代将私钥交换为短期安装令牌。参见 [GitHub App 认证](https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app)。

---

## 12. CLI 参考

```
clem --version                     打印已安装版本
clem update                        下载并安装最新版本
clem init [--backend discord|github]
                                   生成 clem.yaml + CLAUDE.{shared,<agent>}.md 脚手架
clem vault init                    生成 age 密钥对 + .sops.yaml
clem vault set <vault> KEY=value   在保险库中设置密钥
clem vault get <vault> KEY         读取解密的密钥
clem vault list                    列出所有保险库及其密钥（值隐藏）
clem vault delete <vault> [KEY]    删除密钥或整个保险库
clem provision [--remote HOST]     创建 OS 用户、写入 .env、安装服务（需要 root）
clem login [agent...]              作为每个智能体运行 `claude /login`（一次性）
clem start                         启动所有智能体；启用单元，启动看门狗（需要 root）
clem stop                          停止所有智能体；禁用单元以便不重启（需要 root）
clem restart                       停止，然后启动（需要 root）
clem status                        表格：systemd · tmux · 令牌到期 · 最后日志
clem logs <agent>                  跟踪智能体运行器日志
```

### 标志

- `--config <path>` — 覆盖默认的 `clem.yaml` 路径
- `--remote <user@host>` 用于 `provision`/`login` — 通过 SSH 在远程主机上运行

---

## 13. clem.yaml 配置参考

```yaml
project: string             # OS 用户和服务名称前缀
primary_milestone: string   # 可选 — 被 CLAUDE.shared.md 引用

coordination:
  backend: string           # discord（默认）| slack | github
  server_id: string         # Discord 服务器 ID 或 Slack 工作区 ID（github 不使用）
  github_repo: string       # 任务板仓库的 owner/name（backend 为 github 时需要）
  channels:
    general: string         # 频道 ID — 仅 Discord/Slack
    tasks:   string         # Discord/Slack：频道 ID。GitHub：标签（如 clem:todo）
    alerts:  string         # Discord/Slack：频道 ID。GitHub：Issue 编号
    lessons: string         # Discord/Slack：频道 ID。GitHub：Issue 编号

agents:
  <agentkey>:               # 小写；用于 CLI + OS 用户名
    name: string            # Claude + Discord 中的显示名称
    role: string            # 人类可读的角色
    model: string           # 模型 ID（根据提供者，为 Claude 或 Ollama 等名称）
    iteration: duration     # Go 风格时长："30s"、"1m30s"、"2h"（默认 5m）
    vaults: [string]        # 合并到 .env 的保险库名称（后面的保险库优先）
    prompt: string          # 每次会话开始时注入
    web_terminal_port: int  # 可选 — ttyd 端口（1024-65535）用于只读查看
    caveman: off|lite|full|ultra  # 可选 — 安装 caveman 插件（压缩输出 ~75%）；true → ultra（旧版兼容）
    subagent_model: string  # 可选 — Task 工具/Explore/general-purpose 的 CLAUDE_CODE_SUBAGENT_MODEL；默认为 claude-sonnet-4-6；设为 "off" 继承主模型
    provider: string        # 可选 — anthropic（默认）| bedrock | vertex | ollama | openai-compat
    provider_url: string    # provider 为 ollama 或 openai-compat 时需要
    runtime: string         # 可选 — claude-code（默认）| opencode | codex
```

### 运行时（Runtimes）

| `runtime`     | CLI 可执行文件                       | 说明                                                                               |
|---------------|--------------------------------------|------------------------------------------------------------------------------------|
| `claude-code` | `~/.local/bin/claude`                | 默认。Anthropic 原生线格式。最适合云 Claude。                                      |
| `opencode`    | `~/.opencode/bin/opencode`           | 通过 models.dev 原生对接 75+ 提供者。本地模型的工具使用更佳。                       |
| `codex`       | `~/.npm-global/bin/codex`            | OpenAI Codex CLI。支持 ChatGPT OAuth 或 `OPENAI_API_KEY`。                         |

### 提供者（Providers）

| `provider`       | `clem` 写入 `.env` 的额外环境变量                                | 说明                                                       |
|------------------|-------------------------------------------------------------------|------------------------------------------------------------|
| `anthropic`      | 无（默认行为）                                                    | 使用 Claude Code 的 OAuth 或 `ANTHROPIC_API_KEY`           |
| `bedrock`        | `CLAUDE_CODE_USE_BEDROCK=1`                                       | 智能体还需要保险库中的 AWS 凭证                            |
| `vertex`         | `CLAUDE_CODE_USE_VERTEX=1`                                        | 智能体还需要 `GOOGLE_APPLICATION_CREDENTIALS`              |
| `ollama`         | `ANTHROPIC_BASE_URL=<url>` · `ANTHROPIC_MODEL=<model>` · `ANTHROPIC_AUTH_TOKEN=none` | Ollama 原生说 Anthropic API — 不需要代理                   |
| `openai-compat`  | 同 `ollama`                                                       | 需要您自己运行 Anthropic 线翻译器                          |

### 派生名称

- OS 用户：`<project>-<agentkey>`（例如 `myteam-lead`）
- Systemd 服务：`clem-<project>-<agentkey>.service`
- GitHub Issue 观察器：`clem-github-watch-<project>-<agentkey>.service`（仅 github 后端）
- Web 终端：`clem-ttyd-<project>-<agentkey>.service`

---

## 14. 密钥管理

密钥存储在 `secrets.sops.yaml` 中，通过 age 和 sops 加密。该文件可以安全提交。age 私钥（`~/.config/sops/age/keys.txt`）是您唯一必须保留在 git 之外的东西 — 备份它。

`clem provision` 在配置时将密钥解密到每智能体的 `/home/<user>/.env`（模式 0600）。运行器在每次迭代开始时 source 它。密钥在配置后永不离开主机。

每个智能体的 `vaults:` 列表按顺序指定要合并的保险库。后面的保险库覆盖前面的键 — 对共享令牌和每智能体覆盖很有用。

### 常用密钥

- `GH_TOKEN` — GitHub 访问
- `DISCORD_TOKEN` — Discord 机器人（**原始令牌，无 `Bot ` 前缀** — `discord.py` 添加它）
- `SLACK_MCP_XOXP_TOKEN` — Slack 机器人（`xoxb-…`）或用户（`xoxp-…`）令牌
- `SSH_HOST`、`ES_PASSWORD` — 可选，启用 Prefect MCP 服务器
- `WRANGLER_OAUTH_TOKEN` — 可选，启用 Cloudflare Workers MCP

---

## 15. VPS 部署

`clem` 不需要 VPS — 任何 Linux 主机都可以。但对于始终在线的智能体，小型云主机（2-4 GB RAM）便宜且能在笔记本睡眠时保持运行。

### 远程配置流程

```bash
# 在本地机器上，在团队仓库内
clem provision --remote root@<vps-ip> --gh-token ghp_...
clem login --remote root@<vps-ip>
ssh <vps-ip> "cd my-team && clem start && clem status"
```

参见 [docs/hetzner.md](https://github.com/jahwag/clem/blob/main/docs/hetzner.md) 获取 Hetzner 特定的演练（cloud-init、`hcloud` CLI、SSH 配置）。

---

## 16. 故障排除

### `clem provision` 失败，报 `useradd: command not found`
不是 Linux，或缺少核心用户空间。使用标准 Ubuntu/Debian 主机。

### `clem status` 显示 `SYSTEMD=failed`
检查服务：`systemctl status clem-<project>-<agentkey>.service`。常见原因：`.env` 缺失（设置保险库后重新运行 `clem provision`）、每智能体未安装 `clem`（provision 重新安装）、或 MCP 服务器二进制文件不在 PATH 上。

### 智能体未发布到 Discord/Slack
检查 `clem logs <agent>`。运行器记录 MCP 服务器启动。如果 `mcp-discord` 缺失，用 `pipx` 安装（推荐）以便其依赖项生活在隔离的 venv 中：`pipx install git+https://github.com/Bytelope/mcp-discord.git`。避免对 Python MCP 服务器使用 `pip install --break-system-packages` — 智能体服务以 `ProtectHome=read-only` 运行，因此任何后续依赖漂移都无法从沙盒内自愈，MCP 将无法启动。`pipx` venv 将每个 MCP 与系统 Python 状态解耦并能在 `apt upgrade` 后存活。确认机器人已被邀请到服务器。**`DISCORD_TOKEN` 必须是原始令牌**（无 `Bot ` 前缀）；`discord.py` 内部添加它 — 粘贴 `"Bot …"` 会产生 401。对于 Slack：使用机器人令牌（`xoxb-`），而非用户令牌（`xoxp-`）— 用户令牌以您的身份发布，而非机器人。

### 智能体未接收 GitHub 任务
确认 `coordination.backend: github`、`github_repo` 和 `channels.tasks` 已设置。检查观察器：`systemctl status clem-github-watch-<project>-<agent>.service` 和智能体家目录下的 `~/.claude/<agent>-github-watch.log`。观察器需要智能体 `.env` 中的 `GH_TOKEN`。开放 Issue 必须有任务标签且无受让人。启用 `egress:` 后，确保 `api.github.com` 可通过代理访问（`backend: github` 时自动允许）。

### `clem login` 每天持续提示 / `clem status` 每 8 小时翻转为 `EXPIRED`
您可能运行了比 v0.8.4 更旧的 clem。Claude Max *访问*令牌实际上仅持续约 8 小时；Claude Code 会从存储在旁边的刷新令牌自动刷新它。v0.8.4 之前的 `clem status` 显示访问令牌到期时间，v0.8.4 之前的 `NeedsLogin` 以 7 天窗口为门控 — 因此它总是报告"已过期"并训练操作员每天登录。升级到 v0.8.4+；状态现在在存在刷新令牌时显示 `auto-refresh`，仅在实际需要手动 `clem login` 时报告 `missing`。

### 认证智能体 — API 密钥、本地模型或订阅登录
clem 使用您提供的任何凭证运行官方运行时，每个智能体独立认证。根据您的使用选择：
- **`ANTHROPIC_API_KEY`** — 不会每天过期，适合自动化、始终在线或多智能体使用。存储在保险库中（可代理，因此智能体从不读取原始密钥）。
- **本地或非 Anthropic 模型** — 设置 `runtime: opencode`（或将 claude-code 指向 Ollama）以运行本地或第三方模型，无需 Anthropic 认证。
- **交互式 Claude 订阅登录**（`clem login`）— 方便您观看的单个个人智能体。一个 Claude 账户上的多个智能体不可靠：它们的会话在登录刷新时相互干扰，且 `401 Invalid authentication credentials` 失败很常见 — 对于多个智能体，首选 API 密钥或单个非交互式令牌（见下文）。订阅计划也假设普通个人使用，Anthropic 的[使用政策](https://www.anthropic.com/legal/aup)和[消费者条款](https://www.anthropic.com/legal/consumer-terms)仅通过 API 允许自动化或始终在线访问 — 在订阅上运行此类工作负载前请查看。

要以非交互式方式认证订阅，`claude setup-token` 颁发长期（~1年）令牌，您将其暴露为 `CLAUDE_CODE_OAUTH_TOKEN`（将其存储在保险库中并将其添加到智能体的 `reveal_secrets` 以便它落入 `.env`）。Claude Code 静态使用它，因此不受每次会话刷新影响；`clem status` 然后显示 `missing`，因为 clem 仅读取 OAuth 凭证文件，此路径从不创建。

### 令牌实际缺失（`clem status` 显示 `missing`）
重新运行 `sudo clem login <agent>`，或使用 `ANTHROPIC_API_KEY` 或 `CLAUDE_CODE_OAUTH_TOKEN` 非交互式认证（见上文）。`clem status` 仅读取 OAuth 凭证文件，因此按设计对这些非交互式路径显示 `missing`。否则真正缺失的凭证意味着文件被擦除或刷新令牌被服务器端撤销。

### 智能唤醒后什么都不做
Discord：打开任务论坛 — 必须存在带有 `[TODO` 状态的线程。Slack：顶级消息需要 ⏳ 表情反应。GitHub：开放 Issue 必须带有 `clem:todo` 标签。智能体仅处理看板上的内容。

### 在同一主机上配置两次
安全。`useradd` 是幂等的；systemd 单元被覆盖；`.env` 从当前保险库重新生成。现有 Claude OAuth 令牌被保留。

---

## 17. 社区与许可

**社区：** 问题、想法、展示您的团队 — 加入 [ClaudeSync / Clem Discord](https://discord.gg/pR4qeMH4u4)

**许可证：** MIT — 参见 [LICENSE](https://github.com/jahwag/clem/blob/main/LICENSE)

---

## 附录：快速命令速查表

| 操作 | 命令 |
|------|------|
| 查看版本 | `clem --version` |
| 升级 | `sudo clem update` |
| 初始化配置 | `clem init [--backend discord\|github]` |
| 初始化保险库 | `clem vault init` |
| 设置密钥 | `clem vault set <vault> KEY=value` |
| 获取密钥 | `clem vault get <vault> KEY` |
| 列出保险库 | `clem vault list` |
| 删除密钥 | `clem vault delete <vault> [KEY]` |
| 配置主机 | `sudo clem provision [--remote HOST]` |
| 登录认证 | `sudo clem login [agent...]` |
| 启动舰队 | `sudo clem start` |
| 停止舰队 | `sudo clem stop` |
| 重启舰队 | `sudo clem restart` |
| 查看状态 | `clem status` |
| 查看日志 | `clem logs <agent>` |

---

*文档生成时间：2026-07-26，基于 [jahwag/clem](https://github.com/jahwag/clem) 仓库 README。*

---

☕️ 制作不易，请我喝咖啡☕️关注我➕