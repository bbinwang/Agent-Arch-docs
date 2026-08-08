# 附录B 架构决策记录 (ADR)

> **文件**: `docs/wangbin/10-appendix-b.md`  
> **预计 Token**: ~8,000  
> **核心内容**: 关键架构决策的历史、理由与影响

---

## B.1 ADR-001: 选择 LangGraph 作为多 Agent 编排框架

### 状态

**已接受** (Accepted)

### 背景

多 Agent 协作研究需要管理复杂的状态转换和 Agent 交互。需要选择一个编排框架来简化开发。

### 选项

| 选项 | 说明 |
|------|------|
| **LangGraph** | LangChain 官方的状态机框架 |
| AutoGen |微软的多 Agent 框架 |
| CrewAI | 角色驱动的 Agent 框架 |
| 自定义实现 | 自行实现状态机 |

### 决策

选择 **LangGraph** 作为主要的多 Agent 编排框架。

### 理由

1. **生态集成**: 与 LangChain/LangChain Core 无缝集成
2. **状态管理**: 内置 TypedDict 状态管理
3. **条件边**: 支持复杂的条件路由（Human-in-the-loop）
4. **检查点**: 支持状态持久化和恢复
5. **社区活跃**: LangChain 官方维护

### 影响

- **正面**: 简化多 Agent 编排，减少样板代码
- **负面**: 引入 LangGraph 依赖，版本约束 (<0.3)
- **迁移**: 后期添加 AutoGen (AG2) 作为替代实现

### 相关文件

- `multi_agents/agents/orchestrator.py` — ChiefEditorAgent + StateGraph
- `multi_agents/memory/research.py` — ResearchState

---

## B.2 ADR-002: Retriever 插件化架构

### 状态

**已接受** (Accepted)

### 背景

需要支持多种搜索引擎和学术数据库，且允许用户自定义检索器。

### 选项

| 选项 | 说明 |
|------|------|
| **工厂模式 + 注册表** | match-case 动态加载 |
| 抽象基类 + 子类注册 | ABC + __subclasses__ |
| 配置文件驱动 | YAML/JSON 配置 |
| 动态导入 | importlib 动态加载 |

### 决策

选择 **工厂模式 + match-case** 实现检索器注册表。

### 理由

1. **简洁**: match-case 语法清晰
2. **懒加载**: 仅在需要时导入模块
3. **类型安全**: 返回类型明确
4. **可扩展**: 添加新检索器只需添加 case

### 实现

```python
def get_retriever(retriever: str):
    match retriever:
        case "tavily":
            from gpt_researcher.retrievers import TavilySearch
            return TavilySearch
        # ... 20+ 检索器
        case _:
            return None
```

### 影响

- **正面**: 添加新检索器简单，无需修改现有代码
- **负面**: 注册表集中，文件随检索器增多而增大

---

## B.3 ADR-003: 双前端策略

### 状态

**已接受** (Accepted)

### 背景

需要同时支持快速演示和生产级部署的前端。

### 选项

| 选项 | 说明 |
|------|------|
| **双前端** | 静态 HTML + NextJS |
| 仅 NextJS | 统一使用 NextJS |
| 仅静态 | 仅使用 HTML/CSS/JS |

### 决策

选择 **双前端策略**：静态 HTML 用于快速演示，NextJS 用于生产部署。

### 理由

1. **零构建**: 静态前端无需构建步骤，直接打开即用
2. **快速演示**: 用户无需安装 Node.js 即可体验
3. **生产级**: NextJS 提供完整 SPA 体验
4. **嵌入式**: 静态前端可嵌入 iframe

### 影响

- **正面**: 兼顾易用性和功能性
- **负面**: 双前端维护成本
- **未来**: 计划统一为 NextJS

---

## B.4 ADR-004: MCP (Model Context Protocol) 集成

### 状态

**已接受** (Accepted)

### 背景

需要接入外部工具（数据库、API、文件系统）扩展 Agent 能力。

### 选项

| 选项 | 说明 |
|------|------|
| **MCP 标准协议** | 使用 langchain-mcp-adapters |
| 自定义工具接口 | 自行定义工具协议 |
| 直接集成 | 直接调用外部 API |

### 决策

选择 **MCP 标准协议** 集成外部工具。

### 理由

1. **标准化**: MCP 是开放的 AI 工具协议
2. **生态**: 已有大量 MCP 服务器实现
3. **解耦**: 工具与 Agent 解耦，独立开发
4. **安全**: 路径白名单、权限控制

### 实现

```python
# 两阶段方法
class MCPRetriever:
    async def search_async(self, query):
        # 阶段 1: LLM 选择工具
        selected_tools = await self.tool_selector.select_tools(query, all_tools)
        # 阶段 2: LLM 执行研究
        results = await self.mcp_researcher.execute_research(query, selected_tools)
        return results
```

### 影响

- **正面**: 扩展性强，可接入任意 MCP 工具
- **负面**: 依赖 langchain-mcp-adapters

---

## B.5 ADR-005: 弃用 MongoDB 持久化

### 状态

**已接受** (Accepted)

### 背景

早期版本使用 MongoDB 存储报告，但增加了部署复杂度。

### 选项

| 选项 | 说明 |
|------|------|
| **JSON 文件** | 本地 JSON 文件持久化 |
| SQLite | 轻量级关系数据库 |
| 保留 MongoDB | 继续使用 |

### 决策

选择 **JSON 文件** 替代 MongoDB。

### 理由

1. **零依赖**: 无需数据库服务
2. **简单**: 部署只需文件系统
3. **足够**: 报告存储需求简单
4. **快速**: 减少网络 I/O

### 影响

- **正面**: 简化部署，降低运维成本
- **负面**: 不支持复杂查询，并发写入需锁
- **未来**: 可替换为 SQLite 或外部数据库

---

## B.6 ADR-006: 三模型策略 (Fast/Smart/Strategic)

### 状态

**已接受** (Accepted)

### 背景

不同研究任务需要不同能力的 LLM。需要平衡成本和性能。

### 选项

| 选项 | 说明 |
|------|------|
| **三模型** | Fast (快速) + Smart (智能) + Strategic (推理) |
| 单模型 | 统一使用一个模型 |
| 双模型 | 快速 + 智能 |

### 决策

选择 **三模型策略**。

### 理由

1. **成本优化**: 简单任务用小模型
2. **能力分层**: 推理任务用推理模型
3. **降级能力**: Strategic 失败可降级到 Smart

### 模型分工

| 模型 | 默认 | 用途 |
|------|------|------|
| **Fast** | gpt-5.4-mini | 摘要、简单问答 |
| **Smart** | gpt-5.4 | 报告撰写、来源审查 |
| **Strategic** | gpt-5.4 | 查询规划、推理任务 |

### 影响

- **正面**: 平衡成本和性能
- **负面**: 配置复杂度增加

---

## B.7 ADR-007: 异步优先架构

### 状态

**已接受** (Accepted)

### 背景

研究任务涉及大量 I/O 操作（HTTP 请求、LLM 调用），需要高效的并发处理。

### 选项

| 选项 | 说明 |
|------|------|
| **asyncio** | Python 原生异步 |
| ThreadPool | 线程池 |
| 多进程 | multiprocessing |
| 混合 | asyncio + ThreadPool |

### 决策

选择 **asyncio + ThreadPool** 混合模型。

### 理由

1. **I/O 密集**: asyncio 适合 HTTP 请求
2. **阻塞调用**: ThreadPool 处理同步库（如 requests）
3. **性能**: 高并发低开销
4. **生态**: LangChain 原生支持异步

### 实现

```python
# 异步主循环
async def conduct_research(self):
    # 并行搜索
    results = await asyncio.gather(*[
        get_search_results(q, retriever)
        for q in sub_queries
    ])

# 阻塞调用卸载到线程
async def get_search_results(query, retriever):
    search = retriever(query)
    return await asyncio.to_thread(search.search)
```

### 影响

- **正面**: 高并发性能
- **负面**: 调试复杂度增加

---

## B.8 ADR-008: 上下文压缩管道

### 状态

**已接受** (Accepted)

### 背景

研究过程中积累的上下文可能超过 LLM Token 限制，需要压缩策略。

### 选项

| 选项 | 说明 |
|------|------|
| **嵌入相似度过滤** | 基于向量相似度选择最相关块 |
| LLM 摘要 | 使用 LLM 压缩内容 |
| 滑动窗口 | 保留最近 N 个块 |
| 混合 | 嵌入 + LLM |

### 决策

选择 **嵌入相似度过滤** 作为主要压缩策略。

### 理由

1. **成本低**: 无需额外 LLM 调用
2. **速度快**: 向量计算快
3. **效果好**: 语义相关性准确

### 实现

```python
class ContextCompressor:
    async def async_get_context(self, query, max_results=5):
        # 分割文档
        chunks = self._split_documents(self.documents)
        # 计算嵌入相似度
        relevant = await self.embeddings_filter.aquery(query, chunks)
        # 返回最相关的块
        return relevant[:max_results]
```

### 影响

- **正面**: 高效压缩，保持语义相关性
- **负面**: 需要嵌入模型支持

---

## B.9 ADR-009: PromptFamily 继承体系

### 状态

**已接受** (Accepted)

### 背景

不同 LLM 模型（如 Granite）需要不同的提示词格式。

### 选项

| 选项 | 说明 |
|------|------|
| **继承体系** | PromptFamily 基类 + 模型特定子类 |
| 条件判断 | if-else 判断模型 |
| 配置驱动 | 外部模板文件 |

### 决策

选择 **继承体系** 实现提示词管理。

### 理由

1. **开闭原则**: 新模型只需添加子类
2. **类型安全**: 子类可重写特定方法
3. **可维护**: 逻辑封装在类中

### 实现

```python
class PromptFamily:
    # 默认提示词
    def generate_search_queries_prompt(self, ...): ...

class GranitePromptFamily(PromptFamily):
    def pretty_print_docs(self, *args, **kwargs):
        return Granite3PromptFamily.pretty_print_docs(*args, **kwargs)
```

### 影响

- **正面**: 易于扩展新模型
- **负面**: 类层次可能复杂

---

## B.10 ADR-010: 全局限速器单例模式

### 状态

**已接受** (Accepted) — 有争议

### 背景

多个并发研究任务可能同时发起 HTTP 请求，需要全局限速。

### 选项

| 选项 | 说明 |
|------|------|
| **单例模式** | GlobalRateLimiter 类变量单例 |
| 依赖注入 | 通过构造函数传递 |
| 上下文变量 | ContextVar 管理 |

### 决策

选择 **单例模式** 实现全局限速。

### 理由

1. **全局唯一**: 确保所有 WorkerPool 共享同一限速器
2. **简单**: 无需修改大量构造函数
3. **集中管理**: 配置和状态集中

### 争议

- **测试困难**: 单例状态在测试间共享
- **并发安全**: 需要 asyncio.Lock 保护

### 实现

```python
class GlobalRateLimiter:
    _instance: ClassVar['GlobalRateLimiter'] = None
    
    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance
    
    async def wait_if_needed(self):
        async with self.get_lock():
            # 限速逻辑
            ...
```

---

## B.11 ADR 总结

| ADR | 决策 | 状态 | 影响 |
|-----|------|------|------|
| ADR-001 | LangGraph | 已接受 | 多 Agent 编排 |
| ADR-002 | 工厂模式 | 已接受 | Retriever 插件化 |
| ADR-003 | 双前端 | 已接受 | 前端策略 |
| ADR-004 | MCP 集成 | 已接受 | 工具扩展 |
| ADR-005 | JSON 存储 | 已接受 | 简化部署 |
| ADR-006 | 三模型 | 已接受 | 成本优化 |
| ADR-007 | asyncio | 已接受 | 并发性能 |
| ADR-008 | 嵌入压缩 | 已接受 | 上下文管理 |
| ADR-009 | PromptFamily | 已接受 | 提示词管理 |
| ADR-010 | 单例限速 | 已接受 | 全局限速 |

---

> **下一节**: → `10-appendix-c.md` — 附录C 关键算法伪代码与复杂度分析

---

☕️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../res/promotion.jpg)