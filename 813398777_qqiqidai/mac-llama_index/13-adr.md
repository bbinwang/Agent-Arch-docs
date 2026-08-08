# 第 13 章：架构决策记录（ADR）

> 本章记录 LlamaIndex 项目中的重要架构决策，包括决策背景、选项对比、最终选择及其理由。

---

## ADR-001: 选择 Plugin/Integration 架构而非单体包

### 状态
**已接受** (2023 年)

### 背景
LlamaIndex 需要支持大量 LLM 提供商、向量存储、数据源等。如何组织这些集成代码是一个关键架构决策。

### 选项

| 选项 | 描述 | 优点 | 缺点 |
|------|------|------|------|
| **A. 单体包** | 所有集成在一个包中 | 简单、版本统一 | 包体积大、依赖冲突、发布耦合 |
| **B. Plugin/Integration** | 核心包 + 独立集成包 | 按需安装、独立发布、社区贡献清晰 | 版本管理复杂 |
| **C. 子模块** | 核心包内部分子模块 | 统一版本 | 包体积大、耦合高 |

### 决策
选择 **B. Plugin/Integration 架构**。

### 理由
1. **按需安装**: 用户只需安装需要的集成，减少依赖
2. **独立发布**: 集成包可以独立发布，不阻塞核心包
3. **社区贡献**: 清晰的基类和命名约定，降低贡献门槛
4. **避免冲突**: 不同集成的依赖不会相互冲突

### 后果
- 需要维护版本兼容性矩阵
- 需要自动化集成健康检查
- 需要统一的构建和发布流程

---

## ADR-002: 采用 Pydantic v2 作为 Schema 基础

### 状态
**已接受** (2024 年)

### 背景
LlamaIndex 需要强大的数据校验和序列化能力。Pydantic 是 Python 最流行的数据校验库，v2 带来了显著的性能提升和新特性。

### 选项

| 选项 | 描述 | 优点 | 缺点 |
|------|------|------|------|
| **A. Pydantic v1** | 使用 v1 稳定版 | 生态成熟 | 性能较差、新特性缺失 |
| **B. Pydantic v2** | 升级到 v2 | 性能提升 5-50x、新 API | 迁移成本、部分生态未跟进 |
| **C. dataclasses + marshmallow** | 使用标准库 + 第三方 | 无外部依赖 | 功能较弱、需要更多样板代码 |

### 决策
选择 **B. Pydantic v2**。

### 理由
1. **性能**: v2 使用 Rust 核心，性能大幅提升
2. **类型安全**: 更好的类型推断和校验
3. **未来**: v1 将停止维护
4. **生态**: 主流库都已支持 v2

### 后果
- 需要迁移所有 Schema 定义
- 需要更新 `bridge/pydantic.py` 兼容层
- 需要更新文档和示例

---

## ADR-003: 引入 Workflow 系统（外部 llama-index-workflows 包）

### 状态
**已接受** (2024 年)

### 背景
LlamaIndex 需要一个事件驱动的工作流引擎来支持 Agent 和复杂业务流程。

### 选项

| 选项 | 描述 | 优点 | 缺点 |
|------|------|------|------|
| **A. 内置实现** | 在核心包内实现 | 完全控制、无外部依赖 | 开发成本高、维护负担重 |
| **B. 外部包** | 使用 llama-index-workflows | 专注核心、独立迭代 | 外部依赖、版本协调 |
| **C. 集成现有** | 使用 Prefect/Airflow | 成熟方案 | 过重、API 不匹配 |

### 决策
选择 **B. 外部包 llama-index-workflows**。

### 理由
1. **专注核心**: 核心包专注索引和检索
2. **独立迭代**: Workflow 可以独立发布和优化
3. **清晰边界**: 通过明确的接口解耦
4. **复用性**: Workflow 可用于其他项目

### 后果
- 核心包的 `workflow/` 是导出层
- 需要协调版本发布
- 需要清晰的接口文档

---

## ADR-004: PromptMixin/Template Method 模式

### 状态
**已接受** (2023 年)

### 背景
LlamaIndex 的许多组件需要管理 Prompt 模板，需要一个统一的机制来获取和更新这些模板。

### 决策
采用 **PromptMixin** 混入类 + **Template Method** 模式。

### 实现

```python
class PromptMixin(BaseModel):
    def _get_prompts(self) -> PromptDictType:
        """子类实现：返回所有 prompt"""
        return {}

    def _update_prompts(self, prompts: PromptDictType):
        """子类实现：更新 prompt"""
        pass

    def _get_prompt_modules(self) -> PromptMixinType:
        """子类实现：返回子模块"""
        return {}

    def get_prompts(self) -> PromptDictType:
        """获取所有 prompt（包括子模块）"""
        prompts = self._get_prompts()
        for module_name, module in self._get_prompt_modules().items():
            if module is not None:
                module_prompts = module.get_prompts()
                prompts.update({f"{module_name}.{k}": v for k, v in module_prompts.items()})
        return prompts
```

### 理由
1. **统一接口**: 所有组件通过相同方式管理 Prompt
2. **递归获取**: 自动获取子模块的 Prompt
3. **批量更新**: 支持一次性更新所有 Prompt

---

## ADR-005: Storage 层抽象（DocStore/IndexStore/KVStore 分离）

### 状态
**已接受** (2023 年)

### 背景
LlamaIndex 需要持久化多种类型的数据：文档、索引结构、键值对、对话历史。

### 选项

| 选项 | 描述 | 优点 | 缺点 |
|------|------|------|------|
| **A. 统一存储** | 一个 Storage 类处理所有 | 简单 | 耦合高、难以优化 |
| **B. 分离存储** | 按数据类型分离 | 独立优化、可替换 | 接口较多 |
| **C. 外部 ORM** | 使用 SQLAlchemy | 标准化 | 过重、性能开销 |

### 决策
选择 **B. 分离存储**。

### 实现

```python
@dataclass
class StorageContext:
    docstore: BaseDocumentStore      # 文档存储
    index_store: BaseIndexStore      # 索引存储
    vector_stores: Dict[str, BasePydanticVectorStore]  # 向量存储
    graph_store: GraphStore          # 图存储
    property_graph_store: Optional[PropertyGraphStore]  # 属性图存储
```

### 理由
1. **独立优化**: 不同存储可以选择不同后端
2. **可替换**: 每个存储都可以独立替换
3. **清晰边界**: 每个存储职责明确

---

## ADR-006: 采用 uv/hatchling 替代 setuptools/poetry

### 状态
**已接受** (2024 年)

### 背景
LlamaIndex 的 Monorepo 结构需要高效的包管理和构建工具。

### 选项

| 选项 | 描述 | 优点 | 缺点 |
|------|------|------|------|
| **A. setuptools + pip** | 传统工具 | 成熟 | 速度慢、功能有限 |
| **B. poetry** | 现代包管理器 | 功能丰富 | 构建速度一般 |
| **C. uv + hatchling** | 最新工具链 | 极快、现代 | 较新、生态建设中 |

### 决策
选择 **C. uv + hatchling**。

### 理由
1. **速度**: uv 比 pip 快 10-100x
2. **Monorepo 支持**: uv workspace 支持多包管理
3. **标准化**: hatchling 是 PEP 517 标准构建后端
4. **依赖锁定**: `uv.lock` 确保可复现构建

---

## ADR-007: Async 优先设计

### 状态
**已接受** (2023 年)

### 背景
LlamaIndex 的许多操作（LLM 调用、向量检索、文件 IO）是 IO 密集型，异步可以显著提升吞吐量。

### 决策
所有 IO 操作提供 **sync + async 双版本**，内部优先使用 async。

### 实现

```python
class BaseLLM:
    def chat(self, messages) -> ChatResponse: ...      # 同步
    async def achat(self, messages) -> ChatResponse: ...  # 异步

class BaseRetriever:
    def retrieve(self, query) -> List[NodeWithScore]: ...      # 同步
    async def aretrieve(self, query) -> List[NodeWithScore]: ...  # 异步
```

### 同步实现策略

```python
def chat(self, messages):
    return asyncio_run(self.achat(messages))
```

### 理由
1. **性能**: 异步支持高并发
2. **兼容性**: 同步版本兼容非异步代码
3. **渐进式**: 用户可以从同步开始，逐步迁移到异步

---

## ADR-008: Callback → Instrumentation 演进

### 状态
**已接受** (2024 年)

### 背景
LlamaIndex 原有的 Callback 系统功能有限，不支持 Span、结构化事件等现代可观测性需求。

### 选项

| 选项 | 描述 | 优点 | 缺点 |
|------|------|------|------|
| **A. 扩展 Callback** | 在现有基础上扩展 | 向后兼容 | 受限于原有设计 |
| **B. 新 Instrumentation** | 全新设计 | 现代化、功能强 | 迁移成本 |
| **C. 两者并存** | 同时支持 | 平滑迁移 | 维护成本 |

### 决策
选择 **C. 两者并存**（逐步迁移）。

### 实现

```python
# 新代码使用 Instrumentation
import llama_index.core.instrumentation as instrument
dispatcher = instrument.get_dispatcher(__name__)
dispatcher.event(MyEvent())

# 旧代码继续使用 Callback
from llama_index.core.callbacks import CallbackManager
callback_manager = CallbackManager([handler])
```

### 理由
1. **向后兼容**: 现有 Callback 集成继续工作
2. **现代化**: Instrumentation 支持 Span、结构化事件
3. **平滑迁移**: 逐步将 Callback 迁移到 Instrumentation

---

## ADR-009: Monorepo 多包管理策略

### 状态
**已接受** (2023 年)

### 背景
LlamaIndex 包含核心包 + 600+ 集成包，需要有效的管理策略。

### 决策
采用 **Monorepo + uv workspace** 策略。

### 实现

```
llama_index/
├── pyproject.toml          # 根配置（workspace）
├── uv.lock                 # 全局依赖锁定
├── llama-index-core/       # 核心包
├── llama-index-integrations/  # 集成包目录
│   ├── llms/               # 每个子目录包含多个包
│   ├── vector_stores/
│   └── ...
└── llama-dev/              # 开发工具
```

### workspace 配置

```toml
# pyproject.toml
[tool.uv.workspace]
members = [
    "llama-index-core",
    "llama-index-integrations/*/*",
    "llama-index-utils/*",
    "llama-dev",
]
```

### 理由
1. **原子提交**: 核心 + 集成可一次性修改
2. **统一 CI**: 共享构建和测试配置
3. **依赖共享**: 公共依赖统一管理
4. **跨包重构**: 修改基类后所有集成可同步更新

---

## ADR-010: 命名空间包（Namespace Package）设计

### 状态
**已接受** (2023 年)

### 背景
LlamaIndex 的核心包和集成包需要在运行时共享同一个 `llama_index` 命名空间。

### 决策
采用 Python **命名空间包** 机制。

### 实现

```python
# 核心包导入
from llama_index.core.llms import LLM        # llama-index-core 提供
from llama_index.core.indices import VectorStoreIndex

# 集成包导入（注意没有 'core'）
from llama_index.llms.openai import OpenAI   # llama-index-llms-openai 提供
from llama_index.vector_stores.chroma import ChromaVectorStore
```

### 包结构

```
llama_index/                    # 命名空间（无 __init__.py）
├── core/                       # 核心包子目录
│   ├── llms/
│   ├── indices/
│   └── ...
├── llms/                       # LLM 集成子目录
│   ├── openai/
│   ├── anthropic/
│   └── ...
└── vector_stores/              # 向量存储集成子目录
    ├── chroma/
    ├── pinecone/
    └── ...
```

### 理由
1. **无缝拼接**: 核心和集成在运行时共享命名空间
2. **清晰约定**: `core.*` = 核心，`{type}.{provider}` = 集成
3. **灵活安装**: 用户可以选择 Starter 或 Customized 模式

---

## 13.1 ADR 总结

| ADR | 决策 | 关键理由 |
|-----|------|----------|
| **ADR-001** | Plugin/Integration 架构 | 按需安装、独立发布 |
| **ADR-002** | Pydantic v2 | 性能、类型安全 |
| **ADR-003** | 外部 Workflow 包 | 专注核心、独立迭代 |
| **ADR-004** | PromptMixin | 统一 Prompt 管理 |
| **ADR-005** | 分离存储 | 独立优化、可替换 |
| **ADR-006** | uv + hatchling | 速度、标准化 |
| **ADR-007** | Async 优先 | 性能、兼容性 |
| **ADR-008** | Callback + Instrumentation | 向后兼容、现代化 |
| **ADR-009** | Monorepo | 原子提交、统一 CI |
| **ADR-010** | 命名空间包 | 无缝拼接、清晰约定 |

---

## 13.2 小结

本章记录了 LlamaIndex 的 10 个关键架构决策。这些决策共同塑造了 LlamaIndex 当前的架构：

1. **Plugin 架构** 支撑了 600+ 集成生态
2. **Pydantic v2** 提供了类型安全和性能
3. **Workflow 系统** 支撑了 Agent 和复杂流程
4. **分离存储** 提供了灵活的持久化选项
5. **Async 优先** 确保了高性能
6. **命名空间包** 实现了核心和集成的无缝拼接

在下一章中，我们将对关键算法进行伪代码 + 复杂度分析。

---

☕️ 制作不易，请我喝咖啡☕️关注我➕