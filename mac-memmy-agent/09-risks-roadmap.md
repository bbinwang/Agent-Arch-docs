# 09 · 改进建议、风险点与未来规划

## 9.1 当前架构优缺点

### 优点
- **本地优先 + 数据主权**：记忆/配置/App 状态默认全在 `~/.memmy`，扫描入库本地完成，安全边界清晰（本机+显式认证）。
- **跨 Agent 共享记忆的工程闭环**：统一 `SourceAdapter` 契约 + 整会话水位扫描 + 三条写入路径（扫描/采集/手动）最终落同一 `memory.sqlite`，真正实现"Build it once, use it everywhere"。
- **高技术含量的检索引擎**：6 通道混合（向量×3/FTS/pattern/structural）+ RRF 融合 + 相对阈值 + episode 聚合 + MMR + LLM 过滤的四阶管线，外加四层记忆与 RL 式演化（L1→L2→L3→Skill）。
- **可扩展性强**：工具插件化、MCP、Skills 目录化、来源适配器、AgentAdapter 插件清单、Provider 注册表均成体系。
- **容错与可靠性**：fail-open 记忆、显式错误（无假记忆）、scan journal 续跑、Worker 租约+死信+重启自愈、幂等贯穿、多账户隔离。
- **多入口同运行时**：桌面/CLI/API 复用同一 `AgentLoop` 状态机与 `SessionManager`，任务可跨入口接续。

### 缺点 / 技术债
- **超大文件**：`AgentLoop`(~2377 行)、`MemoryService`(~2380 行)、`repositories.ts`(4870 行)、`plugin-algorithms.ts`(检索融合，数千行) 过于庞大，认知与测试成本高。
- **服务层职责过载**：后端 `agent-source-service` 承担发现/扫描/入库/Skill 分发多职责；`loop.ts` 同时管编排/调度/MCP/检查点。
- **两套适配器易混**：`agent-source` 与 `agent-adapter` 职责重叠，命名相近；workbuddy 未用共享 `readJsonlObjects`。
- **散落的安全策略**：SSRF/工作区约束分散在各工具内，缺统一安全策略层。
- **调参门槛**：记忆算法参数极多（capture/reward/l2Induction/l3Abstraction/skill/retrieval），普通用户难调；2000 条向量窗口为不可配常量。
- **重活与 HTTP 同进程**：Memory 无独立 Worker 进程，靠批处理与并发上限缓解对 HTTP 线程的挤占。
- **前端无组件库**：样式散落 CSS，随功能增长可维护性下降。

## 9.2 风险点

| 风险 | 说明 | 缓解 |
| --- | --- | --- |
| 模型/向量质量依赖 | 检索与演化重度依赖 Embedding/LLM 质量；BYOK 模型参差 | 双 LLM 回退（`llm`→`skillLlm`）、本地 Embedding 兜底、LLM 过滤失败回退机械排序 |
| 数据增长与召回延迟 | 长期使用后 `memory.sqlite` 与向量规模增长 | 时间衰减、Dream 整理、2000 条窗口、Tier 截断、MMR 去重 |
| 跨 Agent 配置侵入 | 安装 Hook/插件改写各 Agent 配置文件 | 追加不覆盖、memory slot/Provider 唯一性冲突先确认、移除不删已导入记忆 |
| 隐私与遥测 | 个人记忆敏感 | 本地优先、改进计划显式开关、GA4 仅事件级、secret-store 单独存凭据 |
| 平台/原生模块兼容 | Electron 跨 mac/win + 多原生模块 | `asarUnpack` 解包、`npm rebuild`、Win `MemmyUpdatePrompt.ps1`、版本同步 |
| 失败静默化 | fail-open 可能让用户察觉不到记忆失效 | 显式报错（不返回假记忆）、日志页/`--verbose`/api_logs 可观测 |

## 9.3 改进建议（按优先级）

1. **拆分超大类**：把 `AgentLoop` 的调度/MCP/检查点、`MemoryService` 的编排、`repositories.ts`、`plugin-algorithms.ts` 按职责拆为更小单测友好单元。
2. **统一安全策略层**：把 SSRF/工作区/拒绝模式抽成可组合策略，工具声明式引用。
3. **合并/澄清适配器系统**：明确 `agent-source`(读历史) 与 `agent-adapter`(插件扩展) 边界，统一 workbuddy 到共享 JSONL 助手。
4. **可观测性与调参向导**：把 `--verbose` 的诊断做成 UI 向导，按"提高召回/降噪/多样性"自动推荐参数。
5. **可选独立 Worker**：为 Memory 提供独立 Worker 进程选项，隔离重计算与 HTTP。
6. **前端组件化**：引入轻量 UI 组件层收敛样式。
7. **性能基准与回归**：为检索延迟、Worker 吞吐、扫描吞吐建立基准测试。

## 9.4 技术债清单（优先级）

| 优先级 | 项 |
| --- | --- |
| 高 | 拆分 `AgentLoop`/`MemoryService`/`repositories.ts`/`plugin-algorithms.ts` |
| 高 | 统一工具安全策略层 |
| 中 | 澄清两套适配器系统、统一 workbuddy 读取 |
| 中 | 2000 向量窗口可配化 |
| 中 | 前端样式组件化 |
| 低 | 独立 Worker 进程选项 |
| 低 | 检索/演化参数 UI 向导 |

## 9.5 未来规划（Roadmap）

据官方 README，Memmy 志在**个人记忆基础设施**，范围超越编码 Agent：
- **更多记忆来源**：从 AI 对话扩展到浏览器活动、本地文档，最终更多设备与硬件。
- **团队协作**：规划 Agent-to-Agent 协作，在隐私保护下让团队成员的 AI 助手共享知识。
- 可预见的方向还包括：更丰富的渠道与集成、更强的浏览器/视觉验证能力、记忆质量的持续优化。

---

> 上一节 ← [08 部署运维](./08-deployment-ops.md) ｜ 下一节 → [10 开发者上手指南](./10-developer-guide.md)

---

# ❤️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../res/promotion.jpg)