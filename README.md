# 📚 Agent Architecture Docs

**Agent 项目架构文档合集**

本目录收录了 17 个主流 AI/LLM/Agent 项目的完整技术架构文档，涵盖 RAG、智能体框架、知识库、编码助手等多个方向。每个项目均按照统一标准编写，包含 C4 架构模型、系统流程、核心代码走读、数据模型、API 设计、部署运维、改进建议等章节。

---

## 📂 项目一览

### 🤖 LLM 应用与智能体框架

#### 1. WeKnora（腾讯企业级知识库/RAG）
**GitHub**: [Tencent/WeKnora](https://github.com/Tencent/WeKnora)  
**简介**: 腾讯开源的企业级知识库/RAG 问答引擎，以 RAG 快速问答、ReAct 智能体、Wiki 自动知识库三大核心能力著称。

| 章节 | 一句话介绍 |
|------|------------|
| [第 1 章 项目概述](mac-weknora/01-project-overview.md) | WeKnora v0.7.1 技术架构文档总览，涵盖 RAG 问答、ReAct 智能体、Wiki 自动知识库三大核心能力 |
| [第 2 章 C4 架构模型](mac-weknora/02-c4-architecture.md) | 使用 C4 模型从 Context/Container/Component/Code 四层完整建模 WeKnora 架构 |
| [第 3 章 系统流程与时序图](mac-weknora/03-flows-sequence.md) | 10 个核心业务流程，每个配有 Mermaid 流程图/时序图与 300-500 字说明 |
| [第 4 章 模块结构与依赖分析](mac-weknora/04-module-structure.md) | 完整项目目录树、分层架构、各模块职责与依赖关系 |
| [第 5 章 核心代码深度走读](mac-weknora/05-code-walkthrough.md) | 基于 1,492 个 Go 文件、37.5 万行代码的逐函数深度分析 |
| [第 6 章 数据模型与数据库设计](mac-weknora/06-data-model.md) | ER 图、表结构、索引策略、缓存机制与数据流向 |
| [第 7 章 API 与接口设计](mac-weknora/07-api-design.md) | 238 个 HTTP API 端点（Swagger/OpenAPI），含认证、请求/响应示例 |
| [第 8 章 部署运维与基础设施](mac-weknora/08-deploy-ops.md) | Docker、Helm Chart、CI/CD、监控日志告警、备份恢复方案 |
| [第 9 章 改进建议与未来规划](mac-weknora/09-improvements.md) | 架构优缺点评估、技术债清单与优先级 |
| [第 10 章 增强内容](mac-weknora/10-enhanced-content.md) | 核心组件走读、开发者指南、ADR、算法分析、测试策略 |

---

#### 2. LightRAG（知识图谱增强 RAG）
**GitHub**: [HKUDS/LightRAG](https://github.com/HKUDS/LightRAG)  
**简介**: 简单高效的检索增强生成框架，结合向量与图数据库实现知识图谱增强的 RAG。

| 章节 | 一句话介绍 |
|------|------------|
| [第 1 章 项目概述](mac-LightRAG/01-项目概述.md) | LightRAG v1.5.5 技术架构文档总览 |
| [第 2 章 C4 架构模型](mac-LightRAG/02-C4架构模型.md) | 使用 C4 模型对 LightRAG 进行四层架构建模 |
| [第 3 章 系统流程与时序图](mac-LightRAG/03-系统流程与时序图.md) | 12 个流程图 + 10 个时序图，覆盖核心业务流程 |
| [第 4 章 模块结构与依赖分析](mac-LightRAG/04-模块结构与依赖分析.md) | 完整目录结构、模块职责、依赖关系 |
| [第 5 章 核心代码讲解（上）](mac-LightRAG/05-核心代码讲解-上.md) | lightrag.py / operate.py / pipeline.py 逐函数深度走读 |
| [第 5 章 核心代码讲解（下）](mac-LightRAG/05-核心代码讲解-下.md) | base.py / utils.py / prompt.py / rerank.py 等核心文件分析 |
| [第 6 章 数据模型与数据库设计](mac-LightRAG/06-数据模型与数据库设计.md) | ER 图、13 种存储后端表结构、缓存策略 |
| [第 7 章 API 与接口设计](mac-LightRAG/07-API与接口设计.md) | OpenAPI/Swagger 风格的完整 API 文档 |
| [第 8 章 部署运维与基础设施](mac-LightRAG/08-部署运维与基础设施.md) | Docker/K8s/CI/CD、监控日志告警、备份方案 |
| [第 9 章 改进建议与未来规划](mac-LightRAG/09-改进建议与未来规划.md) | 架构优缺点、风险、技术债、优化建议 |
| [第 10 章 增强内容](mac-LightRAG/10-增强内容.md) | 开发者指南、ADR、算法分析、测试策略 |
| [第 11 章 存储后端详解](mac-LightRAG/11-存储后端详解.md) | 13 种 KV/Vector/Graph/DocStatus 后端实现详细分析 |

---

#### 3. Controllable RAG Agent（可控 RAG 智能体）
**GitHub**: [NirDiamant/Controllable-RAG-Agent](https://github.com/NirDiamant/Controllable-RAG-Agent)  
**简介**: 支持检索结果控制、溯源和干预的 RAG 智能体框架。

| 章节 | 一句话介绍 |
|------|------------|
| [第 1 章 项目概述](mac-Controllable-RAG-Agent/01-project-overview.md) | Controllable RAG Agent 技术架构文档总目录 |
| [第 2 章 C4 架构模型](mac-Controllable-RAG-Agent/02-c4-architecture.md) | 使用 C4 模型对可控 RAG Agent 进行四层架构建模 |
| [第 3 章 系统流程与时序图](mac-Controllable-RAG-Agent/03-flows-and-sequence.md) | 10+ 个 Mermaid 流程图 + 时序图，覆盖核心业务流程 |
| [第 4 章 模块结构与依赖分析](mac-Controllable-RAG-Agent/04-module-structure.md) | 目录树 + 依赖关系图，完整模块职责分析 |
| [第 5 章 核心代码走读](mac-Controllable-RAG-Agent/05-core-code-walkthrough.md) | 所有核心文件逐函数深度走读 |
| [第 6 章 数据模型与数据库设计](mac-Controllable-RAG-Agent/06-data-model.md) | TypedDict 状态、Pydantic Schema、向量存储模型 |
| [第 7 章 API 与接口设计](mac-Controllable-RAG-Agent/07-api-design.md) | 内部 API、LLM 调用接口、Streamlit 接口 |
| [第 8 章 部署运维](mac-Controllable-RAG-Agent/08-deployment.md) | Docker、CI/CD、监控、日志、备份 |
| [第 9 章 改进建议与未来规划](mac-Controllable-RAG-Agent/09-improvements.md) | 架构优缺点、技术债、优化建议、未来路线图 |
| [第 10 章 开发者上手指南](mac-Controllable-RAG-Agent/10-developer-guide.md) | 新贡献者本地运行、调试、测试完整流程 |
| [第 11 章 ADR](mac-Controllable-RAG-Agent/11-adr.md) | 关键架构决策记录与背景理由 |
| [第 12 章 算法分析](mac-Controllable-RAG-Agent/12-algorithms.md) | 核心算法伪代码、时间/空间复杂度、优化建议 |
| [第 13 章 测试策略](mac-Controllable-RAG-Agent/13-testing.md) | 测试金字塔、单元测试、集成测试、评估测试、用例设计 |

---

#### 4. LlamaIndex（LLM 应用开发框架）
**GitHub**: [run-llama/llama_index](https://github.com/run-llama/llama_index)  
**简介**: LLM 应用开发框架，提供数据连接、索引、检索等核心能力。

| 章节 | 一句话介绍 |
|------|------------|
| [第 1 章 项目概述](mac-llama_index/01-project-overview.md) | LlamaIndex v0.14.23 技术架构文档总览 |
| [第 2 章 C4 架构模型](mac-llama_index/02-c4-architecture.md) | 使用 C4 模型从四个抽象层级完整描述 LlamaIndex 系统架构 |
| [第 3 章 系统流程与时序图](mac-llama_index/03-flows-sequence.md) | 10 个核心业务流程，配有 Mermaid 流程图与时序图 |
| [第 4 章 模块结构与依赖分析](mac-llama_index/04-module-structure.md) | 完整目录结构、模块职责、依赖关系图 |
| [第 5 章 核心代码讲解（上）](mac-llama_index/05-code-walkthrough.md) | LLM/Embedding 抽象层、BaseIndex 与 VectorIndex 深度走读 |
| [第 6 章 核心代码讲解（下）](mac-llama_index/06-code-walkthrough-2.md) | Ingestion Pipeline、Workflow、Agent、Storage、Cache 系统分析 |
| [第 7 章 数据模型与数据库设计](mac-llama_index/07-data-model.md) | ER 图、主要表结构、索引、约束、缓存策略 |
| [第 8 章 API 与接口设计](mac-llama_index/08-api-design.md) | 对外及内部 API，含请求/响应示例、认证、限流 |
| [第 9 章 部署运维](mac-llama_index/09-deployment-ops.md) | 部署架构、CI/CD、监控日志告警、备份方案 |
| [第 10 章 改进建议与未来规划](mac-llama_index/10-improvement-risks.md) | 架构评估、技术债清单、优化建议 |
| [第 11 章 组件独立代码走读](mac-llama_index/11-code-walkthrough-per-component.md) | 7 大核心组件独立走读文档 |
| [第 12 章 开发者上手指南](mac-llama_index/12-developer-guide.md) | 本地运行、调试、测试、贡献流程 |
| [第 13 章 ADR](mac-llama_index/13-adr.md) | 重要架构决策历史与理由 |
| [第 14 章 算法分析](mac-llama_index/14-algorithms-complexity.md) | 10 个核心算法伪代码与复杂度分析 |
| [第 15 章 测试策略](mac-llama_index/15-testing-strategy.md) | 测试架构、单元测试、集成测试、覆盖率策略 |

---

#### 5. GPT Researcher（AI 研究助手）
**GitHub**: [assafelovic/gpt-researcher](https://github.com/assafelovic/gpt-researcher)  
**简介**: AI 驱动的研究助手，自动搜索、整理和生成研究报告。

| 章节 | 一句话介绍 |
|------|------------|
| [索引](mac-gpt-researcher/00-index.md) | GPT Researcher v0.14.7 技术架构文档导航 |
| [第 1 章 项目概述](mac-gpt-researcher/01-project-overview.md) | 项目目标、核心价值、技术栈、架构风格 |
| [第 2 章 C4 架构模型](mac-gpt-researcher/02-c4-architecture.md) | 四层架构建模 GPT Researcher |
| [第 3 章 系统流程与时序图](mac-gpt-researcher/03-flows-sequence.md) | 核心业务流程流程图 + 时序图 |
| [第 4 章 模块结构与依赖分析](mac-gpt-researcher/04-module-structure.md) | 完整目录结构、模块职责、依赖关系 |
| [第 5 章 核心代码讲解（上）](mac-gpt-researcher/05-code-walkthrough.md) | Agent + Config + Actions 逐函数深度走读 |
| [第 5 章 核心代码讲解（下）](mac-gpt-researcher/05-code-walkthrough-b.md) | Skills + Retrievers + Scraper 深度分析 |
| [第 6 章 数据模型与数据库设计](mac-gpt-researcher/06-data-model.md) | 向量存储、缓存策略、事务设计 |
| [第 7 章 API 与接口设计](mac-gpt-researcher/07-api-design.md) | RESEARCHER_API 完整协议规范 |
| [第 8 章 部署运维](mac-gpt-researcher/08-deployment.md) | Docker、CI/CD、监控日志告警 |
| [第 9 章 改进建议与未来规划](mac-gpt-researcher/09-improvements.md) | 架构优缺点、技术债、优化建议 |
| [附录 A 开发者上手指南](mac-gpt-researcher/10-appendix-a.md) | 本地运行、调试、测试流程 |
| [附录 B ADR](mac-gpt-researcher/10-appendix-b.md) | 关键架构决策记录 |
| [附录 C 算法分析](mac-gpt-researcher/10-appendix-c.md) | 核心算法伪代码与复杂度分析 |
| [附录 D 测试策略](mac-gpt-researcher/10-appendix-d.md) | 73 个测试用例详解 |
| [附录 E 组件独立走读](mac-gpt-researcher/10-appendix-e.md) | 每个组件独立代码走读文档 |

---

#### 6. Pi（自扩展编码代理）
**GitHub**: [earendil-works/pi](https://github.com/earendil-works/pi)  
**简介**: 自扩展的编码代理命令行工具，通过自然语言与 LLM 协作完成编码任务。

| 章节 | 一句话介绍 |
|------|------------|
| [文档总目录](mac-pi/00-table-of-contents.md) | Pi Agent Harness v0.80.10 完整设计文档索引 |
| [第 1 部分：项目概述](mac-pi/01-project-overview.md) | Pi 是什么、解决什么问题、技术栈、架构风格 |
| [第 2 部分：C4 架构模型](mac-pi/02-c4-architecture.md) | Context/Container/Component/Code 四层建模 |
| [第 3 部分：系统流程与时序图](mac-pi/03-system-flows.md) | 多层级深度流程图 + 时序图 |
| [第 4 部分：模块结构与依赖分析](mac-pi/04-module-structure.md) | monorepo 目录结构、模块职责、依赖关系 |
| [第 5 部分：核心代码讲解](mac-pi/05-code-walkthrough.md) | main.ts 逐文件、逐函数深度走读 |
| [第 6 部分：数据模型与数据库设计](mac-pi/06-data-models.md) | 文件系统优先存储哲学，JSONL 会话文件 |
| [第 7 部分：API 与接口设计](mac-pi/07-api-design.md) | CLI、LLM API、Agent Runtime 三类接口 |
| [第 8 部分：部署运维](mac-pi/08-deployment.md) | GitHub Actions CI/CD、Docker、生产部署 |
| [第 9 部分：改进建议与未来规划](mac-pi/09-improvements.md) | ai/agent-core/coding-agent 三层分离优化建议 |
| [第 10 部分：额外增强内容](mac-pi/10-appendix.md) | Provider Pattern、测试策略、算法分析 |

---

#### 7. Memmy Agent（跨平台会话记忆系统）
**GitHub**: [MemTensor/memmy-agent](https://github.com/MemTensor/memmy-agent)  
**简介**: 跨平台 AI 会话记忆系统，持久化对话上下文，解决每次新会话都要重新自我介绍的痛点。

| 章节 | 一句话介绍 |
|------|------------|
| [架构文档](mac-memmy-agent/index.md) | Memmy-Agent v1.x 完整架构设计文档 |
| [第 1 章 项目概述](mac-memmy-agent/01-overview.md) | Memmy 解决什么问题、核心价值、技术栈 |
| [第 2 章 C4 架构模型](mac-memmy-agent/02-c4-architecture.md) | Context → Container → Component → Code 四层建模 |
| [第 3 章 系统流程与时序图](mac-memmy-agent/03-flows-and-sequences.md) | 10 个核心业务流程，细化到模块/函数级别 |
| [第 4 章 模块结构与依赖分析](mac-memmy-agent/04-module-structure.md) | npm workspaces 根目录结构、模块职责 |
| [第 6 章 数据模型与数据库设计](mac-memmy-agent/06-data-model.md) | 两个独立 SQLite 库：memory.sqlite + desktop-api.sqlite |
| [第 7 章 API 与接口设计](mac-memmy-agent/07-api-design.md) | Memory Runtime API、桌面本地 API、OpenAI 兼容 API |
| [第 8 章 部署运维](mac-memmy-agent/08-deployment-ops.md) | 开发态脚本、生产部署方案 |
| [第 9 章 改进建议与未来规划](mac-memmy-agent/09-risks-roadmap.md) | 本地优先 + 数据主权优势、跨 Agent 兼容性、技术债清单 |
| [第 10 章 开发者上手指南](mac-memmy-agent/10-developer-guide.md) | Node.js >=20、npm、平台准备完整流程 |
| [第 11 章 ADR](mac-memmy-agent/11-adr.md) | 关键架构决策记录与背景理由 |
| [第 12 章 算法与测试策略](mac-memmy-agent/12-algorithms-testing.md) | 记忆检索算法、相似度计算、测试用例 |

---

#### 8. Parlor（端侧多模态 AI 对话助手）
**GitHub**: [parlor-ai/parlor](https://github.com/parlor-ai/parlor)  
**简介**: 完全端侧的多模态 AI 对话助手，支持语音和视觉交互，运行在消费级笔记本上。

| 章节 | 一句话介绍 |
|------|------------|
| [架构文档](mac-parlor/index.md) | Parlor v2.0.0 完全端侧、实时多模态 AI 深度技术文档 |
| [第 1 章 项目概述](mac-parlor/01-overview.md) | Parlor 定位：完全端侧、实时、多模态的 AI 对话助手 |
| [第 2 章 C4 架构模型](mac-parlor/02-c4.md) | Context/Container/Component/Code 四层建模 |
| [第 3 章 系统流程与时序图](mac-parlor/03-flows.md) | 9 个核心业务流程，涵盖正常对话、延迟优化、打断、模式切换等 |
| [第 4 章 模块结构与依赖分析](mac-parlor/04-modules.md) | parlor/ 目录结构、环境变量配置、模块职责 |
| [第 6 章 数据模型与会话状态](mac-parlor/06-data-model.md) | 无状态服务设计：每连接内存会话，连接断开即销毁 |
| [第 7 章 API 与接口设计](mac-parlor/07-api.md) | 2 个 HTTP 端点 + 1 个 WebSocket 端点完整协议规范 |
| [第 8 章 部署运维](mac-parlor/08-deployment.md) | 单机本地部署方案，无 Docker/K8s |
| [第 9 章 改进建议与未来规划](mac-parlor/09-improvements.md) | 架构优缺点、技术债清单、优先级排序 |
| [第 10 章 ADR](mac-parlor/10-adrs.md) | Parlor v2 测量驱动的关键决策记录 |
| [第 11 章 开发者上手指南](mac-parlor/11-developer-guide.md) | 从零到改代码、跑测试、做 benchmark 完整流程 |
| [第 12 章 算法分析](mac-parlor/12-algorithms.md) | 关键算法伪代码 + 时间/空间复杂度 + 优化建议 |
| [第 13 章 测试策略](mac-parlor/13-testing.md) | 端到端 pytest 套件 + 测量驱动 benchmark |

---

#### 9. Sirchmunk（实时数据处理平台）
**GitHub**: [modelscope/sirchmunk](https://github.com/modelscope/sirchmunk)  
**简介**: 从原始数据到自进化智能，实时演进的数据处理平台。

| 章节 | 一句话介绍 |
|------|------------|
| [第 1 章 项目概述](mac-sirchmunk/01-项目概述.md) | Sirchmunk 目标、价值、技术栈、架构风格 |
| [第 2 章 C4 架构模型](mac-sirchmunk/02-C4架构模型.md) | 多层架构建模，搭配 Mermaid 图表 |
| [第 3 章 系统流程与时序图](mac-sirchmunk/03-系统流程与时序图.md) | 8 个核心业务流程，配有流程图/时序图 |
| [第 4 章 模块结构与依赖分析](mac-sirchmunk/04-模块包结构与依赖分析.md) | 完整目录结构树、模块职责、依赖关系图 |
| [第 5 章 核心代码讲解](mac-sirchmunk/05-核心代码讲解.md) | 18 个核心文件逐函数/逐类深度走读 |
| [第 6 章 数据模型与数据库设计](mac-sirchmunk/06-数据模型与数据库设计.md) | ER 图、表结构、索引策略、缓存机制 |
| [第 7 章 API 与接口设计](mac-sirchmunk/07-API与接口设计.md) | 完整 API 文档，含认证、限流、版本控制 |
| [第 8 章 部署运维](mac-sirchmunk/08-部署运维与基础设施.md) | Docker、CI/CD、监控日志告警、备份方案 |
| [第 9 章 改进建议与未来规划](mac-sirchmunk/09-改进建议风险点与未来规划.md) | 架构评估、技术债清单、未来路线图 |
| [第 10 章 额外增强内容](mac-sirchmunk/10-额外增强内容.md) | 代码走读、开发者指南、ADR、算法分析、测试策略 |

---

### 🧠 RAG 与知识库

#### 10. AnythingLLM（本地 LLM 知识库）
**GitHub**: [Mintplex-Labs/anything-llm](https://github.com/Mintplex-Labs/anything-llm)  
**简介**: 本地 LLM 知识库管理工具，支持多模型接入，一键部署私有 RAG。

| 章节 | 一句话介绍 |
|------|------------|
| [目录与阅读指南](mac-anything-llm/00-目录与阅读指南.md) | AnythingLLM v1.15.0 文档导航与版本信息 |
| [第 1 章 项目概述](mac-anything-llm/01-项目概述.md) | 项目背景、目标、技术栈、架构风格 |
| [第 2 章 C4 架构模型](mac-anything-llm/02-C4架构模型.md) | Context/Container/Component/Code 四层建模 |
| [第 3 章 系统流程与时序图](mac-anything-llm/03-系统流程与时序图.md) | 8 个子流程：认证、文档上传、聊天、Agent、模型路由等 |
| [第 4 章 模块结构与依赖分析](mac-anything-llm/04-模块包结构与依赖分析.md) | 完整目录结构、模块职责、依赖关系 |
| [第 5 章 核心代码讲解](mac-anything-llm/05-核心代码讲解.md) | 10 个子模块：服务端启动、认证、向量数据库、AI 适配层等 |
| [第 6 章 数据模型与数据库设计](mac-anything-llm/06-数据模型与数据库设计.md) | ER 图、表结构、索引策略、缓存机制 |
| [第 7 章 API 与接口设计](mac-anything-llm/07-API与接口设计.md) | 完整 API 文档，含认证、限流、版本控制 |
| [第 8 章 部署运维](mac-anything-llm/08-部署运维与基础设施.md) | Docker、CI/CD、监控日志告警、备份方案 |
| [第 9 章 改进建议与未来规划](mac-anything-llm/09-改进建议风险点与未来规划.md) | 架构评估、技术债清单、优化建议 |
| [第 10 章 额外增强内容](mac-anything-llm/10-额外增强内容.md) | 开发者指南、ADR、算法分析、测试策略 |

---

#### 11. RAG Techniques（高级 RAG 技术教程）
**GitHub**: [NirDiamant/RAG_Techniques](https://github.com/NirDiamant/RAG_Techniques)  
**简介**: 涵盖多种高级 RAG 技术的综合教程项目，包括 ReRank、多路召回、查询重写等。

| 章节 | 一句话介绍 |
|------|------------|
| [第 1 章 项目概述](mac-RAG_Techniques/01-project-overview.md) | Advanced RAG Techniques 项目全面概述 |
| [第 2 章 C4 架构模型](mac-RAG_Techniques/02-c4-architecture.md) | 使用 C4 模型对 RAG Techniques 进行四层架构建模 |
| [第 3 章 系统流程与时序图](mac-RAG_Techniques/03-flows-sequence.md) | 10+ 个核心业务流程，每个配有 Mermaid 流程图/时序图 |
| [第 4 章 模块结构与依赖分析](mac-RAG_Techniques/04-module-structure.md) | 目录结构、模块职责、依赖关系 |
| [第 5 章 核心代码讲解](mac-RAG_Techniques/05-core-code-walkthrough.md) | 所有核心文件逐函数、逐类的深度走读 |
| [第 6 章 数据模型与数据库设计](mac-RAG_Techniques/06-data-model.md) | 向量存储模型、缓存策略、事务设计 |
| [第 7 章 API 与接口设计](mac-RAG_Techniques/07-api-design.md) | 对外及内部 API、LLM 接口、数据流接口 |
| [第 8 章 部署运维](mac-RAG_Techniques/08-deployment.md) | Docker、CI/CD、监控方案与运维指南 |
| [第 9 章 改进建议与未来规划](mac-RAG_Techniques/09-improvements.md) | 架构优缺点、技术债清单、优化建议 |
| [第 10 章 开发者上手指南](mac-RAG_Techniques/10-developer-guide.md) | 本地运行、调试、测试、ADR、算法分析、测试策略 |

---

#### 12. Local Deep Research（本地优先 AI 研究助手）
**GitHub**: [LearningCircuit/local-deep-research](https://github.com/LearningCircuit/local-deep-research)  
**简介**: 本地优先的 AI 深度研究助手，无需云端服务，支持加密数据库存储。

| 章节 | 一句话介绍 |
|------|------------|
| [文档索引](mac-local-deep-research/00_INDEX.md) | Local Deep Research v1.10.0 技术文档导航 |
| [第 1 章 项目概述](mac-local-deep-research/01_PROJECT_OVERVIEW.md) | 本地优先 AI 深度研究助手定位与核心价值 |
| [第 2 章 C4 架构模型](mac-local-deep-research/02_C4_ARCHITECTURE.md) | 四层架构建模 Local Deep Research |
| [第 3 章 系统流程与时序图](mac-local-deep-research/03_SYSTEM_FLOWS.md) | 10 个核心流程，配有 Mermaid 流程图、时序图及文字说明 |
| [第 4 章 模块结构与依赖分析](mac-local-deep-research/04_MODULE_STRUCTURE.md) | 模块组织结构、各核心模块职责与依赖关系 |
| [第 5 章 核心代码讲解（上）](mac-local-deep-research/05_CORE_CODE_WALKTHROUGH.md) | 应用入口、搜索系统、策略工厂、LangGraph Agent 分析 |
| [第 5 章 核心代码讲解（下）](mac-local-deep-research/06_CORE_CODE_WALKTHROUGH_2.md) | LLM 提供商抽象层、嵌入与向量存储、内容提取等六大子系统 |
| [第 6 章 数据模型与数据库设计](mac-local-deep-research/07_DATA_MODEL.md) | SQLCipher 加密数据库，每个用户独立加密库 |
| [第 7 章 API 与接口设计](mac-local-deep-research/08_API_DESIGN.md) | REST API v1、研究 API、报告 API 完整协议 |
| [第 8 章 部署运维](mac-local-deep-research/09_DEPLOYMENT.md) | 容器化方案、CI/CD、监控日志体系、GPU 加速部署 |
| [第 9 章 改进建议与未来规划](mac-local-deep-research/10_IMPROVEMENTS.md) | 架构评估、风险识别、可操作改进建议与演进路线 |
| [第 10 章 组件代码走读](mac-local-deep-research/11_CODE_WALKTHROUGH_DETAILED.md) | 五大核心组件深入代码走读分析 |
| [第 12 章 开发者上手指南](mac-local-deep-research/12_DEVELOPER_GUIDE.md) | 从零搭建开发环境、运行调试、贡献代码 |
| [第 13 章 ADR](mac-local-deep-research/13_ADR.md) | 关键架构决策记录与权衡取舍 |
| [第 14 章 算法分析](mac-local-deep-research/14_ALGORITHMS.md) | 核心算法伪代码、时间/空间复杂度、优化建议 |
| [第 15 章 测试策略](mac-local-deep-research/15_TESTING_STRATEGY.md) | 测试金字塔、集成测试、E2E 测试、安全测试 |

---

#### 13. Agentic File Search（AI Agent 文件搜索）
**GitHub**: [PromtEngineer/agentic-file-search](https://github.com/PromtEngineer/agentic-file-search)  
**简介**: AI Agent 驱动的文件搜索工具，支持自然语言查询文件系统。

| 章节 | 一句话介绍 |
|------|------------|
| [项目概述](mac-agentic-file-search/00-项目概述.md) | FsExplorer v0.1.0 项目概述与核心价值 |
| [C4 架构模型](mac-agentic-file-search/01-C4架构模型.md) | 使用 C4 模型对 FsExplorer 进行四层架构建模 |
| [系统流程与时序图](mac-agentic-file-search/02-系统流程与时序图.md) | 12 个 Mermaid 图表，覆盖核心业务流程 |
| [模块结构与依赖分析](mac-agentic-file-search/03-模块结构与依赖分析.md) | 完整项目结构、模块职责、依赖关系、接口契约 |
| [核心代码走读 — 核心层](mac-agentic-file-search/04-核心代码走读-核心层.md) | 核心模块逐函数、逐类深度代码分析 |
| [核心代码走读 — 索引与搜索](mac-agentic-file-search/05-核心代码走读-索引与搜索.md) | pipeline、chunker、schema、metadata、query 分析 |
| [核心代码走读 — 存储与接口](mac-agentic-file-search/06-核心代码走读-存储与接口.md) | base、duckdb、server、exploration_trace、emb 分析 |
| [数据模型与数据库设计](mac-agentic-file-search/07-数据模型与数据库设计.md) | ER 图、表结构、索引策略、缓存机制 |
| [API 与接口设计](mac-agentic-file-search/08-API与接口设计.md) | 完整 API 文档，含认证、限流、版本控制 |
| [部署运维](mac-agentic-file-search/09-部署运维与基础设施.md) | Docker、CI/CD、监控日志告警、备份方案 |
| [改进建议与未来规划](mac-agentic-file-search/10-改进建议与未来规划.md) | 架构优缺点、技术债清单、优化建议 |
| [开发者上手指南](mac-agentic-file-search/11-开发者上手指南.md) | 环境搭建到代码贡献的完整指南 |
| [ADR](mac-agentic-file-search/12-架构决策记录-ADR.md) | 关键架构决策背景、方案选择、权衡分析 |
| [算法分析与复杂度](mac-agentic-file-search/13-算法分析与复杂度.md) | 核心算法伪代码、时间/空间复杂度、优化建议 |
| [测试策略与用例说明](mac-agentic-file-search/14-测试策略与用例说明.md) | 测试金字塔、单元测试、集成测试、用例设计 |

---

#### 14. pyLLMSearch（LLM 驱动搜索）
**GitHub**: [snexus/llm-search](https://github.com/snexus/llm-search)  
**简介**: LLM 驱动的高级 RAG 应用程序，支持智能搜索和问答。

| 章节 | 一句话介绍 |
|------|------------|
| [第 1 章 项目概述](mac-llm-search/01_项目概述.md) | pyLLMSearch 技术架构文档总目录 |
| [第 2 章 C4 架构模型](mac-llm-search/02_C4架构模型.md) | 使用 C4 模型对 pyLLMSearch 进行四层架构建模 |
| [第 3 章 系统流程与时序图](mac-llm-search/03_系统流程与时序图.md) | 10+ 个 Mermaid 图表，覆盖核心业务流程 |
| [第 4 章 模块结构与依赖分析](mac-llm-search/04_模块结构与依赖分析.md) | 完整目录结构、模块职责、依赖关系 |
| [第 5 章 核心代码讲解](mac-llm-search/05_核心代码讲解.md) | 15+ 个核心 Python 文件逐函数深度走读 |
| [第 6 章 数据模型与数据库设计](mac-llm-search/06_数据模型与数据库设计.md) | ER 图、表结构、索引策略、缓存机制 |
| [第 7 章 API 与接口设计](mac-llm-search/07_API与接口设计.md) | FastAPI + FastApiMCP 完整 API 文档 |
| [第 8 章 部署运维](mac-llm-search/08_部署运维与基础设施.md) | Docker、CI/CD、监控日志告警、备份方案 |
| [第 9 章 改进建议与未来规划](mac-llm-search/09_改进建议与未来规划.md) | 架构评估、技术债清单、优化建议 |
| [第 10 章 代码走读文档](mac-llm-search/10_代码走读文档.md) | 5 大核心场景代码走读 |
| [第 11 章 开发者上手指南](mac-llm-search/11_开发者上手指南.md) | 本地运行、调试、测试流程 |
| [第 12 章 ADR](mac-llm-search/12_架构决策记录_ADR.md) | 6 个关键架构决策记录 |
| [第 13 章 算法分析](mac-llm-search/13_算法分析.md) | 5 个核心算法伪代码与复杂度分析 |
| [第 14 章 测试策略](mac-llm-search/14_测试策略.md) | 现有测试 + 建议测试，覆盖率策略 |

---

### 🛠️ 其他项目

#### 15. Awesome LLM Apps（LLM 应用合集）
**GitHub**: [Shubhamsaboo/awesome-llm-apps](https://github.com/Shubhamsaboo/awesome-llm-apps)  
**简介**: 精选的 LLM 应用合集，涵盖各种 AI 应用场景。

| 章节 | 一句话介绍 |
|------|------------|
| [第 1 章 项目概述](mac-awesome-llm-apps/01-项目概述.md) | awesome-llm-apps 仓库全量代码分析（1768 文件 / 503 Python 文件） |
| [第 2 章 C4 架构模型](mac-awesome-llm-apps/02-C4架构模型.md) | 基于 C4 Model 的四层架构建模 |
| [第 3 章 系统流程与时序图](mac-awesome-llm-apps/03-系统流程与时序图.md) | 10 个核心业务流程，每个配 Mermaid 流程图 + 时序图 |
| [第 4 章 模块结构与依赖分析](mac-awesome-llm-apps/04-模块结构与依赖分析.md) | 1768 文件 / 100+ 子项目完整目录结构分析 |
| [第 5 章 核心代码讲解（上）](mac-awesome-llm-apps/05-核心代码讲解（上）.md) | 单 Agent / Starter / RAG 类核心文件逐函数深度走读 |
| [第 5 章 核心代码讲解（下）](mac-awesome-llm-apps/05-核心代码讲解（下）.md) | 多 Agent 团队 / Always-on / MCP / Generative UI 分析 |
| [第 6 章 数据模型与数据库设计](mac-awesome-llm-apps/06-数据模型与数据库设计.md) | SQLite、Chroma/Qdrant、文件型数据、缓存策略 |
| [第 7 章 API 与接口设计](mac-awesome-llm-apps/07-API与接口设计.md) | 对外 API、内部 HTTP API、MCP 协议接口、Pub/Sub 推送 |
| [第 8 章 部署运维](mac-awesome-llm-apps/08-部署运维与基础设施.md) | Docker/K8s/CI-CD、监控日志告警、备份方案 |
| [第 9 章 改进建议与未来规划](mac-awesome-llm-apps/09-改进建议与未来规划.md) | 架构评估、技术债清单、未来路线图 |
| [第 10 章 开发者上手指南](mac-awesome-llm-apps/10-开发者上手指南.md) | 环境准备、本地运行、调试、测试 |
| [第 11 章 ADR 与算法分析](mac-awesome-llm-apps/11-架构决策记录ADR.md) | 架构决策历史、关键算法伪代码、测试策略 |

---

#### 16. Clem（Claude Code 智能体舰队）
**GitHub**: [jahwag/clem](https://github.com/jahwag/clem)  
**简介**: 安全、自托管的 Claude Code 智能体舰队运行框架，每个智能体通过内核级出口防火墙或零秘密凭证代理进行隔离。

| 章节 | 一句话介绍 |
|------|------------|
| [完整文档](mac-clem/clem-documentation.md) | Clem — Continuously Looping Engineering Machines，智能体舰队运行框架完整架构文档 |

---

#### 17. FS Explorer（LlamaIndex 文件系统探索）
**GitHub**: [run-llama/fs-explorer](https://github.com/run-llama/fs-explorer)  
**简介**: 基于 LlamaIndex 的文件系统探索智能体。

| 章节 | 一句话介绍 |
|------|------------|
| [索引与阅读指南](mac-fs-explorer/00-索引与阅读指南.md) | FS Explorer 文档说明、术语表、导航索引 |
| [项目概述](mac-fs-explorer/01-项目概述.md) | 项目目标、核心价值、技术栈清单、架构风格、功能特性 |

---

### 📦 其他开源项目（非 mac 前缀）

#### 18. joyagent-jdgenie（多智能体产品）
**GitHub**: [jd-genie/joyagent-jdgenie](https://github.com/jd-genie/joyagent-jdgenie)  
**简介**: 业界首个开源高完成度轻量化通用多智能体产品。

| 章节 | 一句话介绍 |
|------|------------|
| [Agent Context 设计分析](acer-joyagent/agent-context-design-analysis.md) | JoyAgent Agent Context 模块的设计与实现分析 |
| [RequestID/SessionID 分析](acer-joyagent/agent-context-requestid-sessionid-analysis.md) | JoyAgent 请求 ID 与会话 ID 管理机制分析 |
| [Agent 入口点分析](acer-joyagent/agent-entry-point-analysis.md) | JoyAgent Agent 入口点设计与执行流程分析 |
| [Day4 DataAgent Review](acer-joyagent/joyagent-day4-dataagent-review.md) | JoyAgent Day4 DataAgent 模块评审与分析 |
| [Plan-Solve Architecture](acer-joyagent/joyagent-plan-solve-architecture.md) | JoyAgent Plan-Solve 架构设计与实现分析 |
| [技术文档](acer-joyagent/joyagent-technical-documentation.md) | JoyAgent 完整技术文档与 smolagents 集成分析 |

---

#### 19. AsterMem（分布式记忆系统）
**GitHub**: [Asterove/AsterMem](https://github.com/Asterove/AsterMem)  
**简介**: 分布式 AI 记忆系统，支持多智能体协作场景下的长期记忆存储与检索。

| 章节 | 一句话介绍 |
|------|------------|
| [01-overview](AsterMem架构文档/01-overview.md) | AsterMem 技术架构文档总览 |
| [02-c4-architecture](AsterMem架构文档/02-c4-architecture.md) | 使用 C4 模型对 AsterMem 进行四层架构建模 |
| [03-flows](AsterMem架构文档/03-flows.md) | 核心业务流程流程图与时序图 |
| [04-modules](AsterMem架构文档/04-modules.md) | 模块结构与依赖分析 |
| [05-code-walkthrough](AsterMem架构文档/05-code-walkthrough.md) | 核心代码逐函数深度走读 |
| [06-data-model](AsterMem架构文档/06-data-model.md) | 数据模型与数据库设计 |
| [07-api](AsterMem架构文档/07-api.md) | API 与接口设计 |
| [08-deployment](AsterMem架构文档/08-deployment.md) | Docker、CI/CD、监控日志告警、备份方案 |
| [09-improvements](AsterMem架构文档/09-improvements.md) | 架构优缺点评估、技术债清单、优化建议 |

---

#### 20. scira（Web 浏览 Agent）
**GitHub**: [zaidmukaddam/scira](https://github.com/zaidmukaddam/scira)  
**简介**: Vercel AI SDK + Next.js 驱动的 Web 浏览 Agent，支持智能搜索与信息提取。

| 章节 | 一句话介绍 |
|------|------------|
| [架构文档](scira架构文档/scira-architecture.md) | scira 完整技术架构分析 |

---

#### 21. Vane（端侧 AI 助手）
**GitHub**: [ItzCrazyKns/Vane](https://github.com/ItzCrazyKns/Vane)  
**简介**: 完全本地化的语音助手，支持私有 Agentic 记忆与智能家居控制。

| 章节 | 一句话介绍 |
|------|------------|
| [架构文档](vane架构文档/vane-architecture.md) | Vane 完整技术架构分析 |

---

#### 22. MindSearch（AI 研究助手）
**GitHub**: [InternLM/MindSearch](https://github.com/InternLM/MindSearch)  
**简介**: 基于 InternLM 的 AI 深度研究工具，支持多步推理与信息综合。

| 章节 | 一句话介绍 |
|------|------------|
| [架构文档](mindsearch架构文档/mindsearch-architecture.md) | MindSearch 完整技术架构分析 |

---

#### 23. morphic（AI 聊天界面）
**GitHub**: [miurla/morphic](https://github.com/miurla/morphic)  
**简介**: 基于 Next.js 的 AI 聊天界面，支持多模型接入与智能搜索。

| 章节 | 一句话介绍 |
|------|------------|
| [架构文档](morphic架构文档/morphic-architecture.md) | morphic 完整技术架构分析 |

---

#### 24. markdown-it（Markdown 解析器）
**GitHub**: [markdown-it/markdown-it](https://github.com/markdown-it/markdown-it)  
**简介**: 高性能 Markdown 解析器，支持 CommonMark 规范与插件扩展。

| 章节 | 一句话介绍 |
|------|------------|
| [架构文档](markdown-it架构文档/markdown-it-architecture.md) | markdown-it 完整技术架构分析 |

---

#### 25. qdrant（向量数据库）
**GitHub**: [qdrant/qdrant](https://github.com/qdrant/qdrant)  
**简介**: 高性能开源向量数据库，支持语义搜索、过滤与混合检索。

| 章节 | 一句话介绍 |
|------|------------|
| [架构文档](qdrant架构文档/qdrant-architecture.md) | qdrant 完整技术架构分析 |

---

## 📊 统计信息

| 指标 | 数值 |
|------|------|
| 项目总数 | 25 |
| 文档文件总数 | ~380+ |
| 总文件大小 | ~24 MB |
| 涵盖方向 | RAG、智能体框架、知识库、编码助手、语音助手、文件搜索、研究工具、向量数据库等 |

---

## 🏷️ 项目分类

### LLM 应用与智能体框架（9 个）
- WeKnora · LightRAG · Controllable RAG Agent · LlamaIndex · GPT Researcher · Pi · Memmy Agent · Parlor · Sirchmunk

### RAG 与知识库（5 个）
- AnythingLLM · RAG Techniques · Local Deep Research · Agentic File Search · pyLLMSearch

### 其他开源项目（11 个）
- Awesome LLM Apps · Clem · FS Explorer · joyagent-jdgenie · AsterMem · scira · Vane · MindSearch · morphic · markdown-it · qdrant

---

## 📝 文档标准

每个项目的文档均按照以下统一结构编写：

1. **项目概述** — 目标、价值、技术栈、架构风格
2. **C4 架构模型** — Context/Container/Component/Code 四层建模
3. **系统流程与时序图** — Mermaid 流程图 + 时序图
4. **模块结构与依赖分析** — 目录树 + 依赖关系图
5. **核心代码讲解** — 逐文件、逐函数深度走读
6. **数据模型与数据库设计** — ER 图、表结构、索引策略
7. **API 与接口设计** — 完整 API 文档
8. **部署运维与基础设施** — Docker/CI-CD/监控
9. **改进建议与未来规划** — 架构评估、技术债清单
10. **增强内容** — 开发者指南、ADR、算法分析、测试策略

---

## 🔗 相关资源

- [飞书项目汇总文档](https://my.feishu.cn/docx/YaZVdthNZohjtox0PubcyfH5nif)
- [百度网盘源目录](https://pan.baidu.com/s/1xxx)（请替换为实际链接）

---

> **生成日期**: 2026-08-08  
> **文档版本**: v1.0  
> **维护者**: wangbin

---

☕️ 制作不易，请我喝咖啡☕️关注我➕