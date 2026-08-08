# 第 12 章：开发者上手指南

> 本章为开发者提供完整的本地运行、调试、测试、贡献流程指南。

---

## 12.1 环境准备

### 12.1.1 系统要求

| 项目 | 要求 |
|------|------|
| **操作系统** | macOS / Linux / Windows (WSL2) |
| **Python** | >= 3.10, < 4.0 |
| **包管理器** | uv (推荐) 或 pip |
| **Git** | >= 2.30 |

### 12.1.2 安装 uv

```bash
# macOS / Linux
curl -LsSf https://astral.sh/uv/install.sh | sh

# Windows
powershell -c "irm https://astral.sh/uv/install.ps1 | iex"

# 验证
uv --version
```

### 12.1.3 克隆仓库

```bash
git clone https://github.com/run-llama/llama_index.git
cd llama_index
```

---

## 12.2 本地安装

### 12.2.1 核心包安装

```bash
# 安装核心包（开发模式）
cd llama-index-core
uv sync

# 验证安装
uv run python -c "import llama_index.core; print(llama_index.core.__version__)"
```

### 12.2.2 集成包安装

```bash
# 安装特定 LLM 集成
cd llama-index-integrations/llms/llama-index-llms-openai
uv sync

# 安装特定向量存储集成
cd llama-index-integrations/vector_stores/llama-index-vector-stores-chroma
uv sync
```

### 12.2.3 环境变量配置

```bash
# 创建 .env 文件
cp .env.example .env

# 编辑 .env
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
```

---

## 12.3 运行测试

### 12.3.1 运行核心包测试

```bash
cd llama-index-core
uv run pytest tests/ -x -v

# 运行特定测试
uv run pytest tests/indices/vector_store/test_base.py -v

# 带覆盖率
uv run pytest tests/ --cov=llama_index.core --cov-report=html
```

### 12.3.2 运行集成包测试

```bash
cd llama-index-integrations/llms/llama-index-llms-openai
uv run pytest tests/ -x -v
```

### 12.3.3 使用 llama-dev 测试

```bash
# 运行所有受影响包的测试
llama-dev test --base-ref origin/main --workers 8

# 带覆盖率
llama-dev test --cov --cov-fail-under 80

# 快速失败
llama-dev test --fail-fast
```

### 12.3.4 测试配置

```ini
# pytest.ini 或 pyproject.toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
env_files = [".env"]
```

---

## 12.4 代码风格与 Lint

### 12.4.1 安装 pre-commit

```bash
pre-commit install
```

### 12.4.2 运行所有检查

```bash
# 运行所有 pre-commit 钩子
pre-commit run --all-files

# 仅运行 ruff
pre-commit run ruff --all-files

# 仅运行 mypy
pre-commit run mypy --all-files
```

### 12.4.3 Ruff 配置

```toml
# pyproject.toml
[tool.ruff]
target-version = "py312"
line-length = 88

[tool.ruff.lint]
select = ["E", "F", "I", "N", "W", "UP", "B", "C4", "SIM"]
ignore = ["E501"]
```

### 12.4.4 Mypy 配置

```toml
[tool.mypy]
disallow_untyped_defs = true
ignore_missing_imports = true
plugins = "pydantic.mypy"
```

---

## 12.5 新增一个 LLM 集成

### 12.5.1 步骤概览

1. 创建包目录
2. 配置 pyproject.toml
3. 实现 BaseLLM 子类
4. 编写测试
5. 注册到 LlamaHub

### 12.5.2 创建包

```bash
mkdir -p llama-index-integrations/llms/llama-index-llms-myllm/llama_index/llms/myllm
cd llama-index-integrations/llms/llama-index-llms-myllm
```

### 12.5.3 配置 pyproject.toml

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "llama-index-llms-myllm"
version = "0.1.0"
requires-python = ">=3.10,<4.0"
dependencies = [
    "llama-index-core>=0.14.0,<0.15",
    "myllm-sdk>=1.0",
]

[tool.llamahub]
contains_example = false
import_path = "llama_index.llms.myllm"

[tool.llamahub.class_authors]
MyLLM = "your-github-username"
```

### 12.5.4 实现 LLM 类

```python
# llama_index/llms/myllm/base.py
from llama_index.core.base.llms.base import BaseLLM
from llama_index.core.base.llms.types import (
    ChatMessage, ChatResponse, CompletionResponse, LLMMetadata
)

class MyLLM(BaseLLM):
    model_name: str = "myllm-default"
    
    @property
    def metadata(self) -> LLMMetadata:
        return LLMMetadata(
            context_window=4096,
            num_output=256,
            is_chat_model=True,
            model_name=self.model_name,
        )
    
    @property
    def _model_kwargs(self) -> dict:
        return {"model": self.model_name}
    
    def chat(self, messages, **kwargs) -> ChatResponse:
        # 实现对话逻辑
        response = client.chat(messages)
        return ChatMessage(role="assistant", content=response.text)
    
    def complete(self, prompt, formatted=False, **kwargs) -> CompletionResponse:
        # 实现补全逻辑
        response = client.complete(prompt)
        return CompletionResponse(text=response.text)
    
    # 实现 async 方法...
    
    @classmethod
    def class_name(cls) -> str:
        return "MyLLM"
```

### 12.5.5 编写测试

```python
# tests/test_llms_myllm.py
import pytest
from llama_index.llms.myllm import MyLLM

def test_class_name():
    assert MyLLM.class_name() == "MyLLM"

def test_metadata():
    llm = MyLLM()
    assert llm.metadata.is_chat_model is True
```

---

## 12.6 新增一个 Vector Store 集成

### 12.6.1 实现 VectorStore 类

```python
from llama_index.core.vector_stores.types import (
    BasePydanticVectorStore, VectorStoreQuery, VectorStoreQueryResult
)
from llama_index.core.schema import TextNode

class MyVectorStore(BasePydanticVectorStore):
    stores_text: bool = True
    
    def add(self, nodes):
        # 添加节点到存储
        return [node.node_id for node in nodes]
    
    def delete(self, ref_doc_id, **kwargs):
        # 删除节点
        pass
    
    def query(self, query, **kwargs):
        # 执行向量检索
        return VectorStoreQueryResult(
            nodes=[...],
            similarities=[...],
            ids=[...],
        )
    
    @property
    def client(self):
        return self._client
    
    @classmethod
    def class_name(cls) -> str:
        return "MyVectorStore"
```

---

## 12.7 本地文档构建

```bash
# 安装文档依赖
cd docs
uv sync

# 构建文档
make html

# 实时预览
make watch-docs
# 或
sphinx-autobuild . _build/html --open-browser
```

---

## 12.8 调试技巧

### 12.8.1 启用详细日志

```python
import logging
logging.basicConfig(level=logging.DEBUG)

# 或仅 LlamaIndex
logging.getLogger("llama_index").setLevel(logging.DEBUG)
```

### 12.8.2 使用 LlamaDebugHandler

```python
from llama_index.core.callbacks import LlamaDebugHandler, CallbackManager

debug_handler = LlamaDebugHandler(print_trace_on_end=True)
callback_manager = CallbackManager([debug_handler])

Settings.callback_manager = callback_manager
```

### 12.8.3 使用 MockLLM 测试

```python
from llama_index.core.llms import MockLLM
from llama_index.core import Settings

Settings.llm = MockLLM()  # 返回预设响应
```

### 12.8.4 使用 MockEmbedding 测试

```python
from llama_index.core.embeddings import MockEmbedding

Settings.embed_model = MockEmbedding(embed_dim=1536)
```

---

## 12.9 贡献流程

### 12.9.1 贡献步骤

1. **Fork 仓库**: GitHub 上 Fork `run-llama/llama_index`
2. **创建分支**: `git checkout -b feature/my-feature`
3. **实现功能**: 编写代码 + 测试
4. **运行检查**: `pre-commit run --all-files`
5. **提交**: 遵循 Conventional Commits 规范
6. **创建 PR**: 填写 PR 模板
7. **CI 检查**: 等待 GitHub Actions 通过
8. **代码审查**: 响应审查意见
9. **合并**: 维护者合并

### 12.9.2 Commit 规范

```
feat(core): add new transformation component
fix(llms): resolve OpenAI streaming issue
docs(readme): update installation guide
test(core): add tests for VectorStoreIndex
refactor(ingestion): simplify pipeline logic
chore(deps): bump openai to 1.10
```

### 12.9.3 PR 检查清单

- [ ] 代码通过 `pre-commit run --all-files`
- [ ] 新增功能有测试覆盖
- [ ] 文档已更新
- [ ] Changelog 已添加
- [ ] 向后兼容（或有迁移说明）

---

## 12.10 常见问题

### 12.10.1 安装问题

**Q: `uv sync` 失败**
```bash
# 清除缓存重试
uv cache clean
uv sync
```

**Q: 导入错误 `ModuleNotFoundError`**
```bash
# 确认在正确的目录
cd llama-index-core
uv sync
```

### 12.10.2 测试问题

**Q: 异步测试失败**
```bash
# 确认 pytest-asyncio 配置
# pyproject.toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
```

**Q: 测试超时**
```bash
# 增加超时
uv run pytest tests/ --timeout=120
```

### 12.10.3 LLM 调用问题

**Q: API Key 未设置**
```bash
export OPENAI_API_KEY=sk-...
# 或使用 .env 文件
```

**Q: 速率限制**
```python
from llama_index.core.rate_limiter import RateLimiter
Settings.llm.rate_limiter = RateLimiter(rate_limit=10, time_period=1.0)
```

---

## 12.11 资源链接

| 资源 | 链接 |
|------|------|
| **官方文档** | https://docs.llamaindex.ai |
| **API 参考** | https://developers.llamaindex.ai/python/framework/ |
| **GitHub** | https://github.com/run-llama/llama_index |
| **Discord** | https://discord.gg/dGcwcsnxhU |
| **LlamaHub** | https://llamahub.ai/ |
| **Twitter** | https://x.com/llama_index |

---

## 12.12 小结

本章提供了完整的开发者上手指南：

1. **环境准备**: Python 3.10+, uv, Git
2. **本地安装**: `uv sync` 开发模式安装
3. **运行测试**: pytest + pytest-asyncio
4. **代码风格**: pre-commit + ruff + mypy
5. **新增集成**: LLM/VectorStore/Reader 集成模板
6. **调试技巧**: 日志、Mock、DebugHandler
7. **贡献流程**: Fork → Branch → PR → Review → Merge

在下一章中，我们将生成架构决策记录（ADR）文档。

---

# ❤️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../res/promotion.jpg)