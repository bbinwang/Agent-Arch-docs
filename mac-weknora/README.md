# 📘 WeKnora v0.7.1 — 完整设计文档与代码讲解

> 基于 1,492 个 Go 文件、37.5 万行代码、238 个 API 端点的深度分析

---

## 📋 文档总目录

| 章节 | 文件 | 内容概要 | 规模 |
|------|------|---------|------|
| **第 1 章** | [01-project-overview.md](./01-project-overview.md) | 项目背景、目标、技术栈完整清单、架构风格、功能特性矩阵、非功能性需求 | ~8K tokens |
| **第 2 章** | [02-c4-architecture.md](./02-c4-architecture.md) | C4 四层架构模型（Context/Container/Component/Code），含 Mermaid 图表 + 400+ 字说明/层 | ~25K tokens |
| **第 3 章** | [03-flows-sequence.md](./03-flows-sequence.md) | 10 个核心业务流程（认证/RAG/Agent/Wiki/IM/MCP/数据源/任务队列/可观测性），含 Mermaid 流程图/时序图 | ~40K tokens |
| **第 4 章** | [04-module-structure.md](./04-module-structure.md) | 完整目录树、分层架构图、模块间依赖关系图、职责矩阵 | ~15K tokens |
| **第 5 章** | [05-code-walkthrough.md](./05-code-walkthrough.md) | 逐文件逐函数深度走读（入口/容器/配置/路由/中间件/Handler/Service/Agent/Models/IM/MCP/数据源/基础设施） | ~80K tokens |
| **第 6 章** | [06-data-model.md](./06-data-model.md) | ER 图、表结构详细设计、向量库 Schema、Redis 数据结构、事务设计、数据流向 | ~20K tokens |
| **第 7 章** | [07-api-design.md](./07-api-design.md) | 238 个 API 端点（6 大认证方式）、请求/响应示例、错误码规范、限流策略 | ~30K tokens |
| **第 8 章** | [08-deploy-ops.md](./08-deploy-ops.md) | 部署架构图、Docker 24 服务、Helm Chart、CI/CD 4 workflow、Langfuse 监控、备份恢复 | ~15K tokens |
| **第 9 章** | [09-improvements.md](./09-improvements.md) | 架构优缺点、性能优化建议、安全加固、技术债清单与优先级矩阵、未来路线图 | ~12K tokens |
| **第 10 章** | [10-enhanced-content.md](./10-enhanced-content.md) | 6 个核心组件代码走读、开发者上手指南、10 个 ADR、7 个算法分析、测试策略 | ~35K tokens |

**总计**：~280,000 tokens，10 个 Markdown 文件

---

## 🚀 快速导航

### 按角色阅读

| 角色 | 推荐阅读顺序 |
|------|------------|
| **新人入职** | 第 1 章 → 第 4 章 → 第 10.2 节（开发者指南）→ 第 5 章 |
| **架构师** | 第 2 章 → 第 9 章 → 第 10.3 节（ADR）→ 第 6 章 |
| **后端开发** | 第 5 章 → 第 7 章 → 第 3 章 → 第 8 章 |
| **前端开发** | 第 1 章 → 第 7 章（API 列表）→ 第 3.3/3.4 节（问答流程）|
| **运维/DevOps** | 第 8 章 → 第 9 章 → 第 6 章（数据备份）|
| **安全审计** | 第 7.12 节（限流）→ 第 9.4 节（安全加固）→ 第 6.5 节（事务）|

### 按主题阅读

| 主题 | 相关章节 |
|------|---------|
| Agent 推理 | 第 2.3.2 节 → 第 3.4 节 → 第 5.7 节 → 第 10.1.1 节 |
| RAG 检索 | 第 3.3 节 → 第 5.6.2 节 → 第 10.4.2/10.4.3 节 |
| Wiki 生成 | 第 3.5 节 → 第 5.6.4 节 → 第 10.1.5 节 |
| 多租户安全 | 第 5.4 节（中间件）→ 第 6.5 节（事务）→ 第 9.4 节 |
| 部署运维 | 第 8 章 → 第 10.2 节 |
| API 集成 | 第 7 章 → 第 10.2 节 |

---

## 📊 项目规模统计

| 指标 | 数值 |
|------|------|
| Go 源码文件 | 1,492 个 |
| Go 代码行数 | ~375,000 行 |
| 前端文件 | 464 个 |
| 前端代码行数 | ~182,000 行 |
| Python 文件 | ~62 个 |
| LLM Provider | 29 个 |
| 向量数据库后端 | 12+ |
| IM 渠道适配器 | 9 个 |
| HTTP API 端点 | 238 个 |
| 接口定义 | 48 个 |
| 数据库迁移 | 4 个引擎 |
| 环境变量 | ~200+ |

---

## 🏗️ 技术栈速览

| 类别 | 技术选型 |
|------|---------|
| 后端 | Go 1.26 + Gin + GORM + dig + asynq |
| 前端 | Vue 3 + TypeScript + Pinia + TDesign + Vite |
| 数据库 | PostgreSQL/MySQL/SQLite + pgvector |
| 向量库 | pgvector/Milvus/Qdrant/Weaviate/ES/OpenSearch/Doris/Tencent VectorDB |
| 缓存/队列 | Redis 7.x |
| 可观测性 | Langfuse (OTLP/OTel) |
| 文档解析 | Python + gRPC (MinerU/PaddleOCR-VL) |
| 部署 | Docker Compose / Helm / GitHub Actions |

---

## 📝 文档说明

本文档基于 WeKnora v0.7.1 源代码的**实际分析**生成，所有代码引用、函数签名、文件路径、行数标注均经过验证。文档中的 Mermaid 图表可直接在支持 Mermaid 的 Markdown 渲染器（GitHub、Typora、VS Code）中渲染。

**生成方法**：
1. 全量扫描项目结构（1,492 Go 文件 + 464 前端文件）
2. 并行启动 4 个研究 Agent 读取核心代码
3. 基于实际代码分析撰写文档
4. 交叉验证 Swagger/OpenAPI 规范（238 端点）

---

## 🔗 相关资源

- [WeKnora GitHub](https://github.com/Tencent/WeKnora)
- [WeKnora 官网](https://weknora.weixin.qq.com)
- [WeKnora CHANGELOG](./CHANGELOG.md)
- [WeKnora README](./README.md)
- [API Swagger 文档](./swagger.json)
- [Docker Compose](./docker-compose.yml)
- [Helm Chart](./helm/)

---

> **文档生成时间**：2026-07-26
> **基于版本**：WeKnora v0.7.1
> **分析范围**：全量源代码（1,492 Go 文件 + 464 前端文件 + 62 Python 文件）
