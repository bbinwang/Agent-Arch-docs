<div align="center">

# Sirchmunk 技术架构文档

**从原始数据到自进化智能，实时演进**

[![Python](https://img.shields.io/badge/Python-3.10%2B-3776AB?style=flat-square&logo=python)](https://www.python.org/)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.100%2B-009688?style=flat-square&logo=fastapi)](https://fastapi.tiangolo.com/)
[![Next.js](https://img.shields.io/badge/Next.js-16-000000?style=flat-square&logo=next.js)](https://nextjs.org/)
[![DuckDB](https://img.shields.io/badge/DuckDB-OLAP-FFF000?style=flat-square&logo=duckdb)](https://duckdb.org/)
[![License](https://img.shields.io/badge/License-Apache%202.0-blue?style=flat-square)](LICENSE)

</div>

---

## 📖 文档概述

本文档是对 **Sirchmunk** 项目的全面、深入、极致详细的技术架构分析与代码讲解文档。由资深系统架构师视角出发，覆盖项目的所有核心模块、数据模型、API 设计、部署架构与未来规划。

### 目标读者

| 读者角色 | 推荐阅读章节 |
|---------|-------------|
| **系统架构师** | 01、02、03、09 |
| **后端开发工程师** | 04、05、06、07 |
| **前端开发工程师** | 04（web/ 部分）、05.18 |
| **DevOps / SRE** | 08 |
| **技术经理 / PM** | 01、09、10.3（ADR） |
| **代码评审 / 质量** | 05、09、10.4 |
| **新入职开发者** | 10.2（开发者上手指南）→ 全文 |

### 阅读顺序建议

```
快速了解 → 01-项目概述
架构设计 → 02-C4架构模型 → 03-系统流程与时序图
代码深入 → 04-模块包结构与依赖分析 → 05-核心代码讲解
数据与接口 → 06-数据模型与数据库设计 → 07-API与接口设计
运维部署 → 08-部署运维与基础设施
前瞻规划 → 09-改进建议风险点与未来规划
增强参考 → 10-额外增强内容
```

---

## 📁 文档目录

| 文件 | 主题 | 字数 |
|------|------|------|
| [01-项目概述.md](./01-项目概述.md) | 项目目标、技术栈、架构风格、特性与 NFR | ~15,500 字 |
| [02-C4架构模型.md](./02-C4架构模型.md) | C4 四层架构图（Context/Container/Component/Code） | ~13,800 字 |
| [03-系统流程与时序图.md](./03-系统流程与时序图.md) | 8+ 核心业务流程流程图 + 5 个时序图 | ~25,000 字 |
| [04-模块包结构与依赖分析.md](./04-模块包结构与依赖分析.md) | 目录结构、模块职责、依赖关系图 | ~32,500 字 |
| [05-核心代码讲解.md](./05-核心代码讲解.md) | 逐文件、逐函数深度走读（18 个核心文件） | ~59,000 字 |
| [06-数据模型与数据库设计.md](./06-数据模型与数据库设计.md) | ER 图、表结构、索引、缓存策略 | ~22,800 字 |
| [07-API与接口设计.md](./07-API与接口设计.md) | 全部 API 列表、请求/响应示例、认证限流 | ~22,800 字 |
| [08-部署运维与基础设施.md](./08-部署运维与基础设施.md) | Docker/CICD/监控/日志/备份 | ~18,300 字 |
| [09-改进建议风险点与未来规划.md](./09-改进建议风险点与未来规划.md) | 优缺点、优化建议、技术债、路线图 | ~17,500 字 |
| [10-额外增强内容.md](./10-额外增强内容.md) | 代码走读/上手指南/ADR/算法分析/测试策略 | ~20,500 字 |

**总字数：约 25 万字 | 图表：26+ 个 Mermaid 图 | 50+ 表格**

---

## 🌰 项目速览（一页纸）

### 一句话总结

**Sirchmunk** 是一个基于 **ReAct Agent** 的 **Embedding-Free**（无嵌入）智能搜索与 **自进化知识库** 系统，无需预索引即可直接对原始文件进行实时检索与知识编译。

### 核心差异化

| 维度 | 传统 RAG | Sirchmunk |
|------|---------|-----------|
| 数据准备 | 向量 ETL 管线（小时级） | 零配置、即丢即搜（秒级） |
| 数据新鲜度 | 静态批量重建索引 | 实时自进化索引 |
| 检索精度 | 近似向量匹配 | 确定性 + 上下文混合 |
| 资源消耗 | 高 RAM/CPU（向量 DB） | 极低（ripgrep-all 直接搜索） |
| 可扩展性 | 线性成本增长 | 原生弹性支持 |

### 技术架构要点

- **后端**：Python 3.10+ / FastAPI / DuckDB / ripgrep-all
- **前端**：Next.js 16 / React 19 / TailwindCSS / TypeScript
- **Agent 引擎**：ReAct（Reasoning + Acting）迭代循环
- **检索引擎**：GrepRetriever（ripgrep-all 封装）+ 蒙特卡洛证据采样
- **知识层**：KnowledgeCluster → Cognition Graph（自进化）
- **协议**：REST API + WebSocket + MCP（Model Context Protocol）
- **部署**：Docker 多阶段构建 / GitHub Actions CI/CD

### 关键数字

- **源文件数**：~80+ Python 文件 + ~30+ TypeScript/TSX 文件
- **核心代码量**：~15,000+ 行 Python（不含前端）
- **支持文件格式**：100+（PDF/DOCX/PPTX/XLSX/MD/HTML/JSON/代码/图片 OCR/压缩包等）
- **API 端点**：50+（REST + WebSocket + MCP Tools）
- **数据库表**：DuckDB 持久化（会话/消息/知识索引）

---

## 📐 文档约定

### 术语表

| 术语 | 含义 |
|------|------|
| **ReAct** | Reasoning + Acting，LLM 推理-行动迭代范式 |
| **rga** | ripgrep-all，跨格式全文检索工具 |
| **KnowledgeCluster** | 知识簇，从多证据源蒸馏的高阶知识单元 |
| **EvidenceUnit** | 证据单元，原始文档中的证据引用 |
| **Cognition Layer** | 认知图层，基于知识簇的语义图 |
| **Evolver** | 知识进化器，定期合并/刷新知识簇 |
| **Monte Carlo Sampling** | 蒙特卡洛采样，随机化证据选择策略 |
| **MCP** | Model Context Protocol，模型上下文协议 |

### 图表规范

- 所有图表使用标准 **Mermaid** 代码块（```mermaid），确保可在 GitHub / Typora / Obsidian 中直接渲染
- 每张图表均搭配 **300-500+ 字** 的文字说明
- C4 架构各层图表搭配 **400+ 字** 说明

### 代码引用规范

- 函数引用格式：`module.py → function_name()`
- 类引用格式：`module.py → ClassName`
- 关键代码行以行号标注（基于分析时的实际代码）

### 分析基准

- **分析日期**：2026-07-26
- **项目版本**：`0.0.8+main`
- **代码基准**：`src/sirchmunk/` + `src/sirchmunk_mcp/` + `web/`

---

## 🔗 快速导航

| 需求 | 跳转 |
|------|------|
| 了解项目是什么 | [01-项目概述 - 项目目标](./01-项目概述.md#11-项目目标与核心价值主张) |
| 理解系统架构 | [02-C4架构模型](./02-C4架构模型.md) |
| 理解搜索流程 | [03-系统流程 - 核心搜索流程](./03-系统流程与时序图.md#31-核心搜索流程agentic-search-pipeline) |
| 查找某个模块 | [04-模块包结构与依赖分析](./04-模块包结构与依赖分析.md) |
| 深入某文件代码 | [05-核心代码讲解](./05-核心代码讲解.md) |
| 查看数据库表结构 | [06-数据模型与数据库设计](./06-数据模型与数据库设计.md) |
| 对接 API | [07-API与接口设计](./07-API与接口设计.md) |
| 部署到生产 | [08-部署运维与基础设施](./08-部署运维与基础设施.md) |
| 架构决策背景 | [10.3 - ADR](./10-额外增强内容.md#103-架构决策记录adr) |
| 本地运行调试 | [10.2 - 开发者上手指南](./10-额外增强内容.md#102-开发者上手指南) |

---

## 📄 版权与许可

本文档基于 Sirchmunk 项目源码分析生成，项目版权归 **ModelScope Contributors** 所有，遵循 [Apache License 2.0](LICENSE) 许可。

---

<div align="center">

**文档版本**：v1.0 | **生成日期**：2026-07-26 | **维护者**：架构文档团队

</div>
