# 第 15 章：测试策略与主要测试用例

> 本章描述 LlamaIndex 的测试架构、单元测试策略、集成测试策略、测试目录结构、Fixture 与 Mock 模式、核心测试用例详解、覆盖率策略。

---

## 15.1 测试架构总览

### 15.1.1 测试框架

| 工具 | 用途 | 版本 |
|------|------|------|
| **pytest** | 测试框架 | >= 8.2.1 |
| **pytest-asyncio** | 异步测试 | >= 0.23.7 |
| **pytest-mock** | Mock 工具 | >= 3.14.0 |
| **pytest-cov** | 覆盖率 | ~5.0 |
| **pytest-timeout** | 测试超时 | >= 2.4.0 |
| **pytest-dotenv** | 环境变量 | 0.5.2 |
| **diff-cover** | 增量覆盖率 | >= 9.2.0 |

### 15.1.2 测试配置

```toml
# pyproject.toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
env_files = [".env"]
```

### 15.1.3 测试类型

```
测试金字塔
                    /\
                   /  \
                  / E2E \          ← 端到端测试（少量）
                 /________\
                /          \
               / Integration \    ← 集成测试（中等）
              /______________\
             /                \
            /    Unit Tests     \  ← 单元测试（大量）
           /____________________\
```

---

## 15.2 单元测试策略

### 15.2.1 Mock 工具

LlamaIndex 提供了一套完整的 Mock 工具，用于隔离外部依赖：

| Mock 工具 | 文件 | 用途 |
|-----------|------|------|
| `MockLLM` | `llms/mock.py` | 模拟 LLM 调用 |
| `MockEmbedding` | `tests/indices/vector_store/mock_services.py` | 模拟 Embedding 计算 |
| `patch_llmpredictor_predict` | `tests/mock_utils/mock_predict.py` | 模拟 LLM Predictor |
| `patch_token_splitter_newline` | `tests/mock_utils/mock_text_splitter.py` | 模拟文本切分 |

### 15.2.2 MockLLM 使用

```python
from llama_index.core.llms import MockLLM
from llama_index.core import Settings

# 使用 MockLLM 替代真实 LLM
Settings.llm = MockLLM()

# MockLLM 返回预设响应
response = Settings.llm.complete("Hello")
print(response.text)  # 返回 mock 响应
```

### 15.2.3 MockEmbedding 使用

```python
from tests.indices.vector_store.mock_services import MockEmbedding
from llama_index.core import Settings

# 使用固定维度的 Mock Embedding
Settings.embed_model = MockEmbedding(embed_dim=1536)
```

---

## 15.3 集成测试策略

### 15.3.1 集成测试原则

1. **外部依赖隔离**: LLM API 调用使用 Mock
2. **VectorStore 隔离**: 使用内存模式或 Docker Compose
3. **文件系统隔离**: 使用临时目录
4. **网络隔离**: 默认禁用网络（`no_networking` fixture）

### 15.3.2 Docker Compose 测试

```yaml
# tests/docker-compose.yml
version: '3.8'
services:
  redis:
    image: redis:7
    ports:
      - "6379:6379"
  postgres:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: test
    ports:
      - "5432:5432"
```

### 15.3.3 集成健康检查

`scripts/integration_health_check.py` 自动化评估集成包质量：

```bash
python integration_health_check.py llama-index-integrations/llms/llama-index-llms-openai
# 输出: 健康分数 0.38
```

---

## 15.4 测试目录结构

### 15.4.1 核心包测试结构

```
llama-index-core/tests/
├── __init__.py
├── conftest.py                     # 全局 fixture
├── docker-compose.yml              # 集成测试 Docker
├── ruff.toml                       # 测试专用 lint 配置
├── mock_utils/                     # Mock 工具
│   ├── mock_predict.py
│   ├── mock_text_splitter.py
│   └── ...
├── agent/                          # Agent 测试
│   ├── react/
│   │   └── test_react.py
│   └── workflow/
│       ├── test_base_agent.py
│       ├── test_react_agent.py
│       └── test_function_agent.py
├── base/                           # 基类测试
├── callbacks/                      # 回调测试
├── chat_engine/                    # 对话引擎测试
├── embeddings/                     # Embedding 测试
├── evaluation/                     # 评估测试
├── indices/                        # 索引测试
│   ├── conftest.py
│   ├── vector_store/
│   │   ├── mock_services.py
│   │   └── test_*.py
│   ├── tree/
│   ├── keyword_table/
│   ├── property_graph/
│   └── ...
├── ingestion/                      # 摄入测试
├── llms/                           # LLM 测试
├── memory/                         # 记忆测试
├── node_parser/                    # 节点解析器测试
├── postprocessor/                  # 后处理器测试
├── prompts/                        # Prompt 测试
├── query_engine/                   # 查询引擎测试
├── retrievers/                     # 检索器测试
├── storage/                        # 存储测试
├── text_splitter/                  # 文本切分器测试
└── tools/                          # 工具测试
```

### 15.4.2 测试文件统计

| 类别 | 测试文件数 | 说明 |
|------|-----------|------|
| **indices/** | ~30 | 索引测试（最多） |
| **llms/** | ~10 | LLM 测试 |
| **agent/** | ~10 | Agent 测试 |
| **query_engine/** | ~10 | 查询引擎测试 |
| **其他** | ~129 | 其他模块 |
| **总计** | **~189** | 核心包测试文件 |

---

## 15.5 Fixture 与 Mock 模式

### 15.5.1 全局 Fixture（`conftest.py`）

```python
@pytest.fixture(autouse=True)
def set_env_vars():
    """设置测试环境变量"""
    os.environ["IS_TESTING"] = "1"
    yield
    del os.environ["IS_TESTING"]

@pytest.fixture()
def allow_networking(monkeypatch):
    """允许网络访问（用于集成测试）"""
    monkeypatch.undo()

@pytest.fixture()
def patch_token_text_splitter(monkeypatch):
    """Mock 文本切分器为简单换行切分"""
    monkeypatch.setattr(SentenceSplitter, "split_text", patch_token_splitter_newline)
```

### 15.5.2 常用 Fixture

```python
@pytest.fixture()
def mock_llm():
    """提供 MockLLM 实例"""
    return MockLLM()

@pytest.fixture()
def mock_embed_model():
    """提供 MockEmbedding 实例"""
    return MockEmbedding(embed_dim=1536)

@pytest.fixture()
def sample_documents():
    """提供示例文档"""
    return [
        Document(text="Document 1", metadata={"source": "test1"}),
        Document(text="Document 2", metadata={"source": "test2"}),
    ]
```

### 15.5.3 Mock 模式示例

```python
from unittest.mock import MagicMock, patch

# 模式 1: 使用 MockLLM
def test_query_engine_with_mock():
    Settings.llm = MockLLM()
    response = query_engine.query("test")
    assert response.response is not None

# 模式 2: 使用 monkeypatch
def test_with_monkeypatch(monkeypatch):
    monkeypatch.setattr(
        OpenAI, "chat", MagicMock(return_value=MagicMock(message=MagicMock(content="mock")))
    )

# 模式 3: 使用 pytest-mock
def test_with_mocker(mocker):
    mock_chat = mocker.patch("openai.OpenAI.chat")
    mock_chat.return_value = MagicMock(message=MagicMock(content="mock"))
```

---

## 15.6 核心测试用例详解

### 15.6.1 VectorStoreIndex 测试

```python
# tests/indices/vector_store/test_base.py
@pytest.mark.asyncio
async def test_vector_store_index_build(sample_documents, mock_embed_model):
    """测试索引构建"""
    Settings.embed_model = mock_embed_model
    index = VectorStoreIndex.from_documents(sample_documents)
    assert len(index.index_struct.nodes_dict) == len(sample_documents)

@pytest.mark.asyncio
async def test_vector_store_index_query(sample_documents, mock_embed_model, mock_llm):
    """测试索引查询"""
    Settings.embed_model = mock_embed_model
    Settings.llm = mock_llm
    
    index = VectorStoreIndex.from_documents(sample_documents)
    query_engine = index.as_query_engine()
    response = await query_engine.aquery("test query")
    
    assert response.response is not None
    assert len(response.source_nodes) > 0

@pytest.mark.asyncio
async def test_vector_store_index_insert_delete(sample_documents, mock_embed_model):
    """测试增量插入和删除"""
    Settings.embed_model = mock_embed_model
    index = VectorStoreIndex.from_documents(sample_documents)
    
    # 插入新文档
    new_doc = Document(text="New document")
    index.insert(new_doc)
    assert len(index.index_struct.nodes_dict) > len(sample_documents)
    
    # 删除文档
    index.delete_ref_doc(sample_documents[0].id_)
    assert sample_documents[0].id_ not in index.index_struct.nodes_dict
```

### 15.6.2 ReActAgent 测试

```python
# tests/agent/workflow/test_react_agent.py
@pytest.mark.asyncio
async def test_react_agent_basic():
    """测试 ReActAgent 基本功能"""
    tool = FunctionTool.from_defaults(fn=lambda x: f"Result: {x}")
    agent = ReActAgent(
        name="TestAgent",
        tools=[tool],
        llm=MockLLM(),
    )
    
    result = agent.run("Search for test")
    assert result is not None

@pytest.mark.asyncio
async def test_react_agent_max_iterations():
    """测试最大迭代次数限制"""
    agent = ReActAgent(
        name="TestAgent",
        tools=[],
        llm=MockLLM(),
    )
    
    with pytest.raises(WorkflowRuntimeError, match="Max iterations"):
        agent.run("Complex task", max_iterations=5)
```

### 15.6.3 IngestionPipeline 测试

```python
# tests/ingestion/test_pipeline.py
def test_ingestion_pipeline_basic(sample_documents):
    """测试基础摄入流程"""
    pipeline = IngestionPipeline(
        transformations=[SentenceSplitter(chunk_size=50)],
    )
    nodes = pipeline.run(documents=sample_documents)
    assert len(nodes) > 0

def test_ingestion_pipeline_dedup():
    """测试去重功能"""
    doc = Document(text="Same content")
    pipeline = IngestionPipeline(
        transformations=[SentenceSplitter(chunk_size=50)],
        docstore=SimpleDocumentStore(),
        docstore_strategy=DocstoreStrategy.DUPLICATES_ONLY,
    )
    
    nodes1 = pipeline.run(documents=[doc])
    nodes2 = pipeline.run(documents=[doc])
    
    # 第二次应该被去重
    assert len(nodes2) == 0

def test_ingestion_pipeline_parallel():
    """测试并行摄入"""
    docs = [Document(text=f"Document {i}") for i in range(100)]
    pipeline = IngestionPipeline(
        transformations=[SentenceSplitter(chunk_size=50)],
    )
    
    nodes = pipeline.run(documents=docs, num_workers=4)
    assert len(nodes) > 0
```

### 15.6.4 RetrieverQueryEngine 测试

```python
# tests/query_engine/test_retriever_query_engine.py
@pytest.mark.asyncio
async def test_retriever_query_engine_basic(sample_documents, mock_embed_model, mock_llm):
    """测试基础查询流程"""
    Settings.embed_model = mock_embed_model
    Settings.llm = mock_llm
    
    index = VectorStoreIndex.from_documents(sample_documents)
    query_engine = index.as_query_engine()
    response = await query_engine.aquery("test")
    
    assert response.response is not None
    assert len(response.source_nodes) > 0

@pytest.mark.asyncio
async def test_retriever_query_engine_streaming(sample_documents, mock_embed_model, mock_llm):
    """测试流式查询"""
    Settings.embed_model = mock_embed_model
    Settings.llm = mock_llm
    
    index = VectorStoreIndex.from_documents(sample_documents)
    query_engine = index.as_query_engine(streaming=True)
    response = await query_engine.aquery("test")
    
    # 流式响应
    tokens = []
    for token in response.response_gen:
        tokens.append(token)
    assert len(tokens) > 0
```

### 15.6.5 Storage 测试

```python
# tests/storage/test_docstore.py
def test_docstore_add_get():
    """测试文档存储添加和获取"""
    store = SimpleDocumentStore()
    doc = Document(text="Test document")
    
    store.add_documents([doc])
    retrieved = store.get_document(doc.id_)
    
    assert retrieved is not None
    assert retrieved.text == doc.text

def test_docstore_delete():
    """测试文档存储删除"""
    store = SimpleDocumentStore()
    doc = Document(text="Test document")
    
    store.add_documents([doc])
    store.delete_document(doc.id_)
    
    assert store.get_document(doc.id_) is None

def test_docstore_persist_load(tmp_path):
    """测试持久化和加载"""
    store = SimpleDocumentStore()
    doc = Document(text="Test document")
    store.add_documents([doc])
    
    persist_path = str(tmp_path / "docstore.json")
    store.persist(persist_path)
    
    loaded_store = SimpleDocumentStore.from_persist_path(persist_path)
    retrieved = loaded_store.get_document(doc.id_)
    
    assert retrieved is not None
    assert retrieved.text == doc.text
```

---

## 15.7 覆盖率策略

### 15.7.1 覆盖率配置

```toml
# pyproject.toml
[tool.coverage.run]
omit = [
    "llama_index/core/instrumentation/*",
    "llama_index/core/workflow/*",
    "tests/*",
]

[tool.coverage.report]
fail_under = 80
```

### 15.7.2 增量覆盖率

```bash
# 使用 diff-cover 检查增量覆盖率
diff-cover coverage.xml --compare-branch=origin/main --fail-under=80
```

### 15.7.3 覆盖率豁免

以下模块被豁免（不纳入覆盖率统计）：
- `instrumentation/`: 可观测性系统（由外部包测试）
- `workflow/`: Workflow 引擎（由 llama-index-workflows 包测试）

---

## 15.8 测试最佳实践

### 15.8.1 命名规范

```python
# 测试函数命名
def test_<feature>_<scenario>_<expected>():
    """测试说明"""
    pass

# 示例
def test_vector_store_index_insert_new_document_success():
def test_react_agent_max_iterations_raises_error():
def test_ingestion_pipeline_dedup_skips_duplicates():
```

### 15.8.2 AAA 模式

```python
def test_example():
    # Arrange（准备）
    doc = Document(text="Test")
    pipeline = IngestionPipeline(transformations=[SentenceSplitter()])
    
    # Act（执行）
    nodes = pipeline.run(documents=[doc])
    
    # Assert（断言）
    assert len(nodes) > 0
    assert nodes[0].text == "Test"
```

### 15.8.3 异步测试

```python
import pytest

@pytest.mark.asyncio
async def test_async_function():
    """异步测试"""
    result = await some_async_function()
    assert result is not None
```

---

## 15.9 CI 测试流程

### 15.9.1 PR 触发流程

```
PR 创建/更新
  → build_package.yml (构建检查)
  → lint.yml (代码风格)
  → core-typecheck.yml (类型检查)
  → unit_test.yml (单元测试)
  → coverage_check.yml (覆盖率)
  → 全部通过 → 可合并
```

### 15.9.2 测试命令

```bash
# 运行所有测试
uv run pytest tests/ -x -v

# 运行特定模块
uv run pytest tests/indices/ -v

# 带覆盖率
uv run pytest tests/ --cov=llama_index.core --cov-report=html

# 并行测试
uv run pytest tests/ -n auto

# 超时控制
uv run pytest tests/ --timeout=120
```

---

## 15.10 测试常见问题

### 15.10.1 异步测试失败

**问题**: `RuntimeError: Event loop is closed`

**解决**: 确认 `pytest-asyncio` 配置正确
```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
```

### 15.10.2 Mock 不生效

**问题**: Mock 未正确应用

**解决**: 确认 patch 路径正确
```python
# 错误: patch 了错误的位置
mocker.patch("llama_index.core.llms.OpenAI.chat")

# 正确: patch 实际使用的位置
mocker.patch("openai.OpenAI.chat.completions.create")
```

### 15.10.3 测试污染

**问题**: 测试间相互影响

**解决**: 使用 fixture 隔离状态
```python
@pytest.fixture(autouse=True)
def reset_settings():
    """每个测试前重置 Settings"""
    Settings.llm = None
    Settings.embed_model = None
    yield
```

---

## 15.11 小结

本章描述了 LlamaIndex 的测试策略：

1. **测试框架**: pytest + pytest-asyncio + pytest-mock
2. **Mock 工具**: MockLLM + MockEmbedding + monkeypatch
3. **测试结构**: 189 个测试文件，覆盖所有核心模块
4. **测试类型**: 单元测试 + 集成测试 + 健康检查
5. **覆盖率**: pytest-cov + diff-cover（增量覆盖）
6. **CI 集成**: GitHub Actions 自动运行测试
7. **最佳实践**: AAA 模式、异步测试、Fixture 隔离

---

## 15.12 全文总结

本技术架构文档对 LlamaIndex v0.14.23 进行了全面、深入、极致详细的分析，共 15 章：

| 章节 | 核心内容 |
|------|----------|
| **01 项目概述** | 目标、技术栈、架构风格、功能特性 |
| **02 C4 架构** | Context/Container/Component/Code 四层模型 |
| **03 流程时序** | 10 个核心业务流程的流程图/时序图 |
| **04 模块结构** | 目录树、模块职责、依赖关系 |
| **05 代码走读(上)** | LLM/Embedding/Index/Retriever/QueryEngine |
| **06 代码走读(下)** | Ingestion/Workflow/Agent/Storage/Callback/Prompt/Schema |
| **07 数据模型** | ER 图、表结构、缓存策略、序列化 |
| **08 API 设计** | 7 大核心 API、请求/响应、认证、限流 |
| **09 部署运维** | 部署架构、CI/CD、监控、日志、备份 |
| **10 改进建议** | 优缺点、性能优化、安全加固、技术债 |
| **11 组件走读** | 7 大核心组件独立走读 |
| **12 开发者指南** | 环境、运行、测试、调试、贡献 |
| **13 ADR** | 10 份架构决策记录 |
| **14 算法复杂度** | 10 个核心算法伪代码 + 复杂度分析 |
| **15 测试策略** | 测试架构、用例、覆盖率 |

**LlamaIndex 的核心竞争力**:
1. 极致的抽象层次（5 行代码到深度定制）
2. 庞大的集成生态（600+）
3. 统一的抽象接口
4. 事件驱动的 Agent/Workflow
5. 企业级可观测性
6. 活跃的社区

**文档生成信息**:
- 项目: LlamaIndex v0.14.23
- 分析日期: 2026-07-26
- 文档位置: `docs/wangbin/`
- 文件数: 16 个 Markdown 文件
- 总字数: ~310K 字
- Mermaid 图表: 30+ 个

---

☕️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../res/promotion.jpg)