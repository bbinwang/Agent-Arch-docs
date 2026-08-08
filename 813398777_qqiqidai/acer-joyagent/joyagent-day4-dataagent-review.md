# JoyAgent Day4 学习资料 — DataAgent 与整体复盘

> 目标：掌握 DataAgent / DGP 协议，理解完整系统架构

---

## 一、DataAgent 总览

JoyDataAgent 是京东开源的端到端"智能数据查询与分析"框架，在 BirdSQL 测试集上达到 **75.74% 准确率**（84 支团队中排名第 4，领先字节跳动 DataAgent 和 IBM）。

### 三大核心能力

| 能力 | 说明 | 关键技术 |
|------|------|----------|
| **DGP 数据治理** | 表设计规范 + 字段语义管理 | DataAgentModelConfig 配置体系 |
| **NL2SQL 智能查询** | 自然语言转 SQL，自适应不同表类型 | TableRAG 两阶段选表选字段 + 3 阶段 LLM Pipeline |
| **自动诊断分析** | 4 阶段迭代分析框架 | smolagents CodeAgent + 7 种统计洞察 |

### 技术栈

```
Java Spring Boot (genie-backend:8080)
    ↕ HTTP
Python FastAPI  (genie-tool:1601)
    ↕
Qdrant (向量库) + Elasticsearch (字段值索引) + JDBC (数据查询)
```

---

## 二、DGP 协议 (Data Governance Protocol)

DGP 是 DataAgent 的数据治理基石，确保 LLM 能准确理解数据语义。

### 三大支柱

```
┌─────────────────────────────────────────────────────────┐
│                    DGP 协议                              │
├──────────────────────┬──────────────────────────────────┤
│  1. 数据治理与挖掘    │  2. 数据血缘治理    │  3. 语义对齐与指标预织 │
│     (已完成)         │    (进行中)         │     (进行中)         │
└──────────────────────┴──────────────────────────────────┘
```

### 2.1 数据治理与挖掘（已完成）

**表设计原则：**
- 事实表与指标表不可混用
- 增量表与全量表不可混用

**字段设计原则：**
- 避免字段歧义（如 `amount` 需明确是"订单金额"还是"支付金额"）
- 标注时间点是"时间点"还是"时间段"

**字段值设计原则：**
- 文档化枚举值语义（如 `status: 1=待付款, 2=已付款, 3=已取消`）

**配置实现** — `DataAgentModelConfig.java`：

```yaml
autobots:
  data-agent:
    model-list:
      - name: "order_detail"
        type: "table"                # table 或 sql
        business-prompt: "订单明细表..."   # 业务描述
        ignore-fields: "id,created_at"    # 忽略字段
        default-recall-fields: "order_id,user_id,amount"  # 默认召回字段
        analyze-suggest-fields: "amount"  # 分析建议字段
        analyze-forbid-fields: "password" # 分析禁止字段
        sync-value-fields: "status"       # 同步枚举值字段
        column-alias-map:                # 字段别名映射
          amount: "订单金额"
```

### 2.2 数据血缘治理（进行中）

通过 SQL AST 解析数据仓库脚本，识别字段、表、处理算子之间的血缘关系，构建知识图谱，结合语义富化用于 RAG 检索。

### 2.3 语义对齐与指标预织（进行中）

跨维度语义归一化、多定义冲突消解、从指标口径和语义口径预织表模型，用于检索时的精确 SQL 约束。

---

## 三、完整系统架构图

```
┌──────────────────────────────────────────────────────────────────────┐
│                         用户前端 (React :3000)                        │
└──────────────────────────────┬───────────────────────────────────────┘
                               │ SSE / REST
                               ▼
┌──────────────────────────────────────────────────────────────────────┐
│                    Java 后端 (Spring Boot :8080)                      │
│                                                                      │
│  ┌─────────────────────┐   ┌───────────────────────────────────┐    │
│  │ DataAgentController │   │ GenieController (通用Agent入口)     │    │
│  │   /data/*           │   │   /AutoAgent                      │    │
│  └─────────┬───────────┘   └──────────────┬────────────────────┘    │
│            │                               │                         │
│            ▼                               ▼                         │
│  ┌─────────────────────┐   ┌───────────────────────────────────┐    │
│  │ DataAgentService    │   │ AgentHandlerFactory               │    │
│  │  - 查询编排         │   │  ├─ PlanSolveHandler (deepThink=1)│    │
│  │  - Schema 富化      │   │  └─ ReactHandler    (deepThink=0)│    │
│  └──────┬──────────────┘   └──────────────┬────────────────────┘    │
│         │                                   │                         │
│         │         ┌─────────────────────────┤                         │
│         │         │                         │                         │
│         ▼         ▼                         ▼                         │
│  ┌────────────┐ ┌──────────────┐ ┌────────────────────┐             │
│  │TableRag    │ │Nl2SqlService │ │ PlanningAgent      │             │
│  │Service     │ │              │ │   ↕                │             │
│  └─────┬──────┘ └──────┬───────┘ │ ExecutorAgent(s)   │             │
│        │               │         │   ↕                │             │
│        │               │         │ SummaryAgent       │             │
│        │               │         └────────────────────┘             │
│        │               │                                             │
│  ┌─────┴───────────────┴──────────────────────────────────┐         │
│  │              HTTP Client (OkHttp)                       │         │
│  └───────────────────────┬────────────────────────────────┘         │
│                          │ POST /v1/tool/*                          │
└──────────────────────────┼─────────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────────────┐
│                    Python 工具服务 (FastAPI :1601)                     │
│                                                                      │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌────────────┐  │
│  │ TableRAGAgent│ │ NL2SQLAgent  │ │AutoAnalysis  │ │ DeepSearch │  │
│  │              │ │              │ │    Agent     │ │            │  │
│  │ ┌──────────┐ │ │ ┌──────────┐ │ │ ┌──────────┐ │ └────────────┘  │
│  │ │ Jieba    │ │ │ │ Rewrite  │ │ │ │smolagents│ │                 │
│  │ │ 关键词   │ │ │ │ Query    │ │ │ │CodeAgent │ │                 │
│  │ ├──────────┤ │ │ ├──────────┤ │ │ ├──────────┤ │                 │
│  │ │ Qdrant   │ │ │ │ Think    │ │ │ │GetData   │ │                 │
│  │ │ 向量召回 │ │ │ │ 思考     │ │ │ │DataTrans │ │                 │
│  │ ├──────────┤ │ │ ├──────────┤ │ │ │Insight   │ │                 │
│  │ │ ES       │ │ │ │ SQL      │ │ │ │SaveIns   │ │                 │
│  │ │ 字段值   │ │ │ │ 生成     │ │ │ │FinalAns  │ │                 │
│  │ ├──────────┤ │ │ └──────────┘ │ │ └──────────┘ │                 │
│  │ │ Column   │ │ │              │ │              │                  │
│  │ │ Filter   │ │ │              │ │ insights.py  │                  │
│  │ └──────────┘ │ │              │ │ data_model   │                  │
│  └──────┬───────┘ └──────┬───────┘ └──────┬───────┘                   │
│         │                │                │                           │
│  ┌──────┴────────────────┴────────────────┘                           │
│  │         Qdrant    Elasticsearch    JDBC                          │
│  │         (:6333)   (:9200)          (业务数据库)                    │
│  └───────────────────────────────────────────────────────────────────│
└──────────────────────────────────────────────────────────────────────┘
```

---

## 四、核心代码精讲

### 4.1 DataAgentService — 查询编排核心

**文件**: `genie-backend/src/main/java/com/jd/genie/service/DataAgentService.java`

```
webChatQueryData(req)              ← SSE 入口
  │
  ├─ getBaseNl2SqlReq(query)       ← 构建基础请求（模型列表、日期、schema 元数据）
  │
  ├─ enrichNl2Sql(req)             ← 调用 TableRAG 召回相关表/字段
  │     └─ tableRagService.tableRag()
  │           └─ HTTP POST → /v1/tool/table_rag
  │
  └─ nl2SqlService.runNL2SQLSse()  ← 调用 NL2SQL 生成 SQL
        └─ HTTP POST → /v1/tool/nl2sql
              └─ nl2sqlQueryData()  ← 后处理：替换表名、解析列、执行SQL、结构化结果
```

**关键设计**: 如果 TableRAG 返回空，回退到加载全部 schema（兜底策略）。

### 4.2 DataAgentController — API 端点

**文件**: `genie-backend/src/main/java/com/jd/genie/controller/DataAgentController.java`

| 端点 | 方法 | 用途 |
|------|------|------|
| `POST /data/chatQuery` | SSE 流式 | 主查询入口 |
| `POST /data/apiChatQuery` | 同步 | 非流式查询 |
| `POST /data/vectorRecall` | 直接调用 | 向量召回调试 |
| `POST /data/esRecall` | 直接调用 | ES 召回调试 |
| `GET /data/allModels` | 列表 | 所有注册模型 |
| `GET /data/previewData` | 预览 | 模型数据预览 |
| `POST /data/testQuery` | 执行 | SQL 测试执行 |

### 4.3 DataAgentConfig — 配置体系

**文件**: `genie-backend/src/main/java/com/jd/genie/config/data/DataAgentConfig.java`

```
DataAgentConfig (prefix: autobots.data-agent)
  ├── agentUrl          → Python 工具服务地址
  ├── modelList         → List<DataAgentModelConfig> (每张表的配置)
  ├── qdrantConfig      → Qdrant 向量库配置
  ├── dbConfig          → JDBC 数据库配置
  └── esConfig          → Elasticsearch 配置

DataAgentConstants
  ├── QDRANT_COLLECTION = "genie_model_schema"
  └── ES_INDEX = "genie_model_column_value"

DataAgentInitRunner (CommandLineRunner)
  └── 启动时初始化 Qdrant Collection + ES Index + 模型元数据
```

### 4.4 NL2SQL 完整流程

**Java 侧**: `genie-backend/.../service/Nl2SqlService.java`
**Python 侧**: `genie-tool/genie_tool/tool/nl2sql.py`

```
                         NL2SQL 完整流程
                               │
            ┌──────────────────┴──────────────────┐
            │        并行阶段 (asyncio.gather)      │
            │                                      │
     ┌──────┴──────┐                      ┌────────┴────────┐
     │ Rewrite     │                      │ Column Filter   │
     │ 查询改写    │                      │ (Rank)          │
     │ rewrite_    │                      │ 两阶段 LLM      │
     │ prompt      │                      │ 选表 + 选列     │
     └──────┬──────┘                      └────────┬────────┘
            │                                      │
            └──────────────────┬───────────────────┘
                               │
                    ┌──────────┴──────────┐
                    │ Schema 格式化        │
                    │ m_schema_format()   │
                    └──────────┬──────────┘
                               │
                    ┌──────────┴──────────┐
                    │ Think 思考阶段       │
                    │ 查询分解 + 字段映射  │
                    │ (SSE 流式输出)       │
                    └──────────┬──────────┘
                               │
                    ┌──────────┴──────────┐
                    │ SQL 生成             │
                    │ @@分隔多SQL,         │
                    │ ##分隔query+sql      │
                    └──────────┬──────────┘
                               │
                               ▼
                    Java 后处理：
                    1. 替换 modelCode → 实际表名
                    2. SqlParserUtils 解析列/过滤条件
                    3. JdbcDataProvider 执行 SQL
                    4. 结构化 → ChatQueryData (维度列/度量列/图表配置)
```

**NL2SQL Prompt 策略**（`genie-tool/genie_tool/prompt/nl2sql.yaml`）：

| 阶段 | Prompt | 核心规则 |
|------|--------|----------|
| 改写 | `rewrite_prompt` | 最小改动原则，补充缺失信息 |
| 思考 | `think_prompt` | 分解问题 → 匹配字段 → 推导逻辑 |
| 生成 | `nl2sql_prompt` | 不用 JOIN、用字段 ID 不用名称、非聚合加 DISTINCT |

### 4.5 TableRAG — 两阶段选表选字段

**文件**: `genie-tool/genie_tool/tool/table_rag/table_rag.py`

```
TableRAG 完整 Pipeline
         │
         ▼
  ┌──────────────┐
  │ Jieba 分词   │ ─── get_jieba_queries() 提取名词/动词关键词
  └──────┬───────┘
         │
    ┌────┴────────────────────────────┐
    │                                 │
    ▼                                 ▼
┌────────────┐                 ┌────────────┐
│ Qdrant     │                 │ ES         │
│ 向量召回    │                 │ 字段值召回  │
│ (列名检索)  │                 │ (枚举值)    │
│            │                 │            │
│ + LLM 生成  │                 │ + LLM 生成  │
│   额外关键词 │                 │   额外关键词 │
└─────┬──────┘                 └─────┬──────┘
      │                               │
      └──────────┬────────────────────┘
                 │
          ┌──────┴──────┐
          │ 合并 & 重排  │
          │ qd_merge_   │
          │ rerank()    │
          │ 累加分数去重  │
          └──────┬──────┘
                 │
          ┌──────┴──────┐
          │ Top-K 选择   │
          │ 按 model 分组│
          │ 取 top-k 表  │
          │ 每表 top-k 列 │
          └──────┬──────┘
                 │
          ┌──────┴──────────────────────┐
          │ ColumnFilterModule (LLM)    │
          │                             │
          │ Stage 1: 表过滤              │
          │   表数 < 3 时跳过            │
          │   LLM 判断表相关性           │
          │                             │
          │ Stage 2: 列过滤              │
          │   每表单独 LLM 判断          │
          │   返回 relatedFlag + 列索引  │
          └─────────────────────────────┘
```

**Retriever** (`retriever.py`) 是底层检索适配器：
- `retrieve_schema()` → 调用 Qdrant 向量检索列名
- `retrieve_cell()` → 调用 ES 检索字段值
- `qd_es_merge()` → 将 ES 字段值合并到 Qdrant schema 作为 fewShot 示例

### 4.6 AutoAnalysisAgent — 自动诊断分析

**文件**: `genie-tool/genie_tool/tool/auto_analysis.py`

```
AutoAnalysisAgent
         │
         ▼
  get_schema()  ← 获取表元数据
         │
         ▼
  create_agent()  ← 创建 smolagents.CodeAgent
  ├── OpenAIServerModel (LLM)
  ├── Python 解释器
  └── 5 个自定义工具：
       ├── GetDataTool      → 调用 Java NL2SQL 获取数据
       ├── DataTransTool    → 数据变换 (率/排名/增量/均值差)
       ├── InsightTool      → 7 种统计洞察分析
       ├── SaveInsightTool  → 保存分析结论
       └── FinalAnswerTool  → 返回最终结果
         │
         ▼
  4 阶段迭代分析框架：
  ┌────────────────────────────────────────────┐
  │ Phase 1: 描述性分析                         │
  │   分布、趋势、缺失值                         │
  ├────────────────────────────────────────────┤
  │ Phase 2: 异常检测                           │
  │   Z-score、IQR、STL 分解                    │
  ├────────────────────────────────────────────┤
  │ Phase 3: 相关性洞察                         │
  │   Spearman、交叉分析、显著性检验             │
  ├────────────────────────────────────────────┤
  │ Phase 4: 预测性分析                         │
  │   ARIMA、SHAP 归因、聚类、因果关系           │
  └────────────────────────────────────────────┘
```

### 4.7 归因分析组件

#### insights.py — 7 种统计洞察类型

| 洞察类型 | 类名 | 检测内容 | 核心算法 |
|----------|------|----------|----------|
| 突出首项 | `OutstandingFirstInsightType` | 维度中最高值 | 幂律拟合 + p-value |
| 突出末项 | `OutstandingLastInsightType` | 维度中最低值 | 负向异常检测 |
| 归因分析 | `AttributionInsightType` | 占比 >50% 的主导值 | 份额计算 |
| 均匀性 | `EvennessInsightType` | 均匀分布 | Shannon 熵 |
| 趋势 | `TrendInsightType` | 线性趋势 | 线性回归 |
| 变化点 | `ChangePointInsightType` | 拐点/变化点 | 峰值检测 + 统计检验 |
| 相关性 | `CorrelationInsightType` | 数值维度间关联 | Pearson 相关系数 |

每个类型都有 `_check()` (前置条件) 和 `_from_data()` (计算) 方法，输出 `impact` (重要性权重) 和 `significance` (统计置信度)。

#### data_model.py — 数据模型

```
Column (字段)
  ├── is_series (是否时间序列)
  └── is_number (是否数值)

Measure (度量)
  ├── type: quantity / ratio
  ├── aggregation: sum/avg/count/...
  └── extend_type: original / rank / rate / sub_avg / increase

DataModel (核心数据容器)
  ├── DataFrame + Measure + 可分析列
  └── 自动检测列类型

SiblingGroup (数据子空间)
  ├── DataModel + 过滤条件 + 下钻列
  └── impact: 相对重要性

SiblingGroupContainer
  ├── 排序的 SiblingGroup 列表
  └── 按 impact 阈值过滤
  └── constract_from_data_model(): 生成所有可能的子空间用于穷举分析
```

#### analysis_tool.py — 5 个分析工具

| 工具 | 功能 | 关键逻辑 |
|------|------|----------|
| `GetDataTool` | 从后端获取数据 | 调用 NL2SQL pipeline，合并多查询结果 |
| `DataTransTool` | 数据变换 | rate(占比)、rank(排名)、increase(增量)、sub_avg(均值差) |
| `InsightTool` | 洞察分析 | 选择 7 种洞察之一，在 DataFrame + breakdown + measure 上执行 |
| `SaveInsightTool` | 保存结论 | 将 data + insight text + 分析过程 存入 AnalysisContext |
| `FinalAnswerTool` | 返回结果 | 汇总所有 insights + 摘要 |

---

## 五、端到端流程

### 5.1 数据查询流程

```
用户提问: "上个月各品类的销售额排名"
    │
    ▼
DataAgentController.chatQuery()              [Java, SSE]
    │
    ▼
DataAgentService.webChatQueryData()
    │
    ├──▶ TableRagService.tableRag()          ──▶ Python TableRAGAgent
    │      │                                      │ Jieba 提取 "上月/品类/销售额/排名"
    │      │                                      │ Qdrant 召回 schema 列 (表名+字段名)
    │      │                                      │ ES 召回字段值 (品类枚举)
    │      │                                      │ 合并 + Top-K 选表选列
    │      │                                      │ LLM 精细过滤列
    │      │◄─────────────────────────────────────┘
    │      │  返回: [{modelCode, columns: [...]}]
    │      │
    │      └── 构建 NL2SQLReq (含召回 schema)
    │
    └──▶ Nl2SqlService.runNL2SQLSse()        ──▶ Python NL2SQLAgent
           │                                      │ 并行: 改写查询 + 排序列
           │                                      │ Think: 分解 + 字段映射
           │                                      │ 生成: SELECT ... GROUP BY ...
           │◄─────────────────────────────────────┘
           │  返回: SQL + query 对
           │
           └── 后处理:
                1. modelCode → 实际表名
                2. 解析 SQL 列/过滤条件
                3. JdbcDataProvider 执行 SQL
                4. 结构化结果 → 维度列 + 度量列 + 图表配置
                    │
                    ▼
              SSE 流式返回前端
```

### 5.2 自动分析流程

```
用户任务: "分析最近一个月销售下滑的原因"
    │
    ▼
AutoAnalysisAgent.run()
    │
    ├── get_schema()  ──▶ 获取表元数据
    │
    └── smolagents CodeAgent 执行:
         │
         ├── Phase 1: GetDataTool ──▶ "查一下最近一个月每日销售额"
         │     └── 回调 Java apiChatQuery ──▶ NL2SQL pipeline ──▶ 返回 DataFrame
         │
         ├── Phase 1: InsightTool ──▶ TrendInsight ──▶ 发现下降趋势
         │     └── SaveInsightTool ──▶ 保存: "销售额日均下降 5%"
         │
         ├── Phase 2: GetDataTool ──▶ "按品类拆分销售额"
         │     └── InsightTool ──▶ AttributionInsight ──▶ 发现某品类占比骤降
         │
         ├── Phase 3: DataTransTool ──▶ 计算增量
         │     └── InsightTool ──▶ ChangePointInsight ──▶ 定位变化点
         │
         ├── Phase 4: InsightTool ──▶ CorrelationInsight ──▶ 相关性分析
         │     └── SaveInsightTool ──▶ 保存归因结论
         │
         └── FinalAnswerTool ──▶ 汇总所有 insights + 摘要
              │
              ▼
         上传分析报告文件
```

---

## 六、文件索引速查表

### Java 后端

| 文件 | 路径 | 职责 |
|------|------|------|
| DataAgentController | `genie-backend/.../controller/DataAgentController.java` | REST API 端点 |
| DataAgentService | `genie-backend/.../service/DataAgentService.java` | 查询编排核心 |
| Nl2SqlService | `genie-backend/.../service/Nl2SqlService.java` | NL2SQL HTTP 桥接 + SQL 后处理 |
| TableRagService | `genie-backend/.../service/TableRagService.java` | TableRAG HTTP 桥接 |
| SchemaRecallService | `genie-backend/.../service/SchemaRecallService.java` | 向量/ES 直接召回 (调试) |
| DataAgentConfig | `genie-backend/.../config/data/DataAgentConfig.java` | 配置绑定 |
| DataAgentModelConfig | `genie-backend/.../config/data/DataAgentModelConfig.java` | 每表元数据配置 |
| DataAgentConstants | `genie-backend/.../config/data/DataAgentConstants.java` | 常量 (Collection/Index 名) |
| DataAgentInitRunner | `genie-backend/.../config/DataAgentInitRunner.java` | 启动初始化 |
| NL2SQLReq | `genie-backend/.../data/dto/NL2SQLReq.java` | NL2SQL 请求 DTO |
| DataAgentChatReq | `genie-backend/.../model/req/DataAgentChatReq.java` | 聊天请求 DTO |

### Python 工具服务

| 文件 | 路径 | 职责 |
|------|------|------|
| NL2SQLAgent | `genie-tool/genie_tool/tool/nl2sql.py` | NL2SQL 3 阶段 Pipeline |
| TableRAGAgent | `genie-tool/genie_tool/tool/table_rag/table_rag.py` | TableRAG 两阶段选表选列 |
| ColumnFilterModule | `genie-tool/genie_tool/tool/table_rag/table_column_filter.py` | LLM 表/列过滤 |
| Retriever | `genie-tool/genie_tool/tool/table_rag/retriever.py` | 底层检索适配器 |
| QdrantRecall | `genie-tool/genie_tool/tool/table_rag/qdrant_recall.py` | Qdrant 客户端封装 |
| AutoAnalysisAgent | `genie-tool/genie_tool/tool/auto_analysis.py` | 自动诊断分析 Agent |
| insights | `genie-tool/genie_tool/tool/analysis_component/insights.py` | 7 种统计洞察 |
| data_model | `genie-tool/genie_tool/tool/analysis_component/data_model.py` | 数据模型定义 |
| analysis_tool | `genie-tool/genie_tool/tool/analysis_component/analysis_tool.py` | 5 个分析工具 |
| schema_data | `genie-tool/genie_tool/tool/analysis_component/schema_data.py` | HTTP 辅助函数 |
| protocal | `genie-tool/genie_tool/model/protocal.py` | 请求模型定义 |
| context | `genie-tool/genie_tool/model/context.py` | AnalysisContext |
| nl2sql.yaml | `genie-tool/genie_tool/prompt/nl2sql.yaml` | NL2SQL Prompt 模板 |
| table_rag.yaml | `genie-tool/genie_tool/prompt/table_rag.yaml` | TableRAG Prompt 模板 |
| analysis.yaml | `genie-tool/genie_tool/prompt/analysis.yaml` | 分析 Prompt 模板 |
| tool.py (API) | `genie-tool/genie_tool/api/tool.py` | FastAPI 路由注册 |

---

## 七、关键设计模式总结

| 模式 | 体现 | 优势 |
|------|------|------|
| **两阶段选表选列** | TableRAG: 粗召回(Jieba+向量+ES) → 精排序(LLM) | 兼顾召回率和精确度 |
| **3 阶段 LLM Pipeline** | NL2SQL: Rewrite → Think → SQL | 分步降低复杂度 |
| **并行执行** | NL2SQL: asyncio.gather 并行 Rewrite 和 Rank | 减少总延迟 |
| **SSE 流式** | 全链路 SSE: Think 过程实时可见 | 用户体验好 |
| **兜底策略** | TableRAG 为空时加载全量 schema | 保证可用性 |
| **CodeAgent 工具编排** | smolagents CodeAgent + 5 个自定义工具 | 灵活的分析能力 |
| **穷举子空间** | SiblingGroupContainer 生成所有子空间 | 不遗漏分析视角 |
| **配置驱动** | DataAgentModelConfig 每表独立配置 | 业务语义精确管理 |

---

## 八、与 JoyAgent 通用框架的关系

DataAgent 是 JoyAgent 多 Agent 框架的**垂直领域应用**：

```
JoyAgent 通用框架
├── Plan-Solve 模式 (PlanningAgent → ExecutorAgent → SummaryAgent)
├── React 模式 (ReActAgent: Think-Act)
├── 工具体系 (FileTool, CodeTool, ReportTool, DeepSearchTool)
├── MCP 工具扩展
└── SSE 流式通信

         +  DataAgent 领域扩展
              ├── DGP 协议 (数据治理配置)
              ├── TableRAG (选表选字段)
              ├── NL2SQL (自然语言转 SQL)
              ├── AutoAnalysis (自动诊断分析)
              └── 归因分析组件 (7 种洞察)
```

DataAgent 中的 `DataAnalysisTool` 被注册为 JoyAgent 的通用工具之一，可以在 Plan-Solve 和 React 模式下被 ExecutorAgent 调用。

---

*整理日期: 2025-05-04 | 代码分支: data_agent*

---

☕️ 制作不易，请我喝咖啡☕️关注我➕