# 附录A 开发者上手指南

> **文件**: `docs/wangbin/10-appendix-a.md`  
> **预计 Token**: ~8,000  
> **核心内容**: 本地运行、调试、测试、贡献流程

---

## A.1 环境准备

### A.1.1 系统要求

| 要求 | 最低 | 推荐 |
|------|------|------|
| Python | 3.11 | 3.12+ |
| Node.js | 18 | 20+ |
| RAM | 4GB | 8GB+ |
| 磁盘 | 2GB | 5GB+ |

### A.1.2 工具安装

```bash
# Python
# 使用 pyenv 管理 Python 版本
pyenv install 3.12.x
pyenv local 3.12.x

# Poetry (推荐)
curl -sSL https://install.python-poetry.org | python3 -

# Node.js (前端开发)
# 使用 nvm
nvm install 20
nvm use 20

# Docker (可选)
# 安装 Docker Desktop
```

---

## A.2 本地运行

### A.2.1 快速开始

```bash
# 1. 克隆仓库
git clone https://github.com/assafelovic/gpt-researcher.git
cd gpt-researcher

# 2. 创建虚拟环境
python -m venv .venv
source .venv/bin/activate  # Linux/Mac
# .venv\Scripts\activate   # Windows

# 3. 安装依赖
pip install -r requirements.txt

# 4. 配置环境变量
cp .env.example .env
# 编辑 .env 文件，添加 API Key

# 5. 启动服务
python main.py
```

### A.2.2 环境变量配置

```bash
# .env 文件
OPENAI_API_KEY=sk-your-openai-key
TAVILY_API_KEY=tvly-your-tavily-key

# 可选
LANGCHAIN_API_KEY=xxx          # LangSmith 追踪
GOOGLE_API_KEY=xxx             # 图像生成
ANTHROPIC_API_KEY=xxx          # Claude 模型

# 自定义 LLM
FAST_LLM=openai:gpt-4o-mini
SMART_LLM=openai:gpt-4o
STRATEGIC_LLM=openai:gpt-4o

# 自定义检索器
RETRIEVER=tavily
# 或多重检索器
RETRIEVER=tavily,duckduckgo,google
```

### A.2.3 使用 Poetry

```bash
# 安装依赖
poetry install

# 激活环境
poetry shell

# 运行
poetry run python main.py
```

### A.2.4 使用 Docker

```bash
# 构建并启动
docker-compose up --build

# 后台运行
docker-compose up -d

# 查看日志
docker-compose logs -f gpt-researcher

# 停止
docker-compose down
```

---

## A.3 前端开发

### A.3.1 静态前端

```bash
# 直接打开
open frontend/index.html

# 或启动简单 HTTP 服务器
cd frontend
python -m http.server 3000
```

### A.3.2 NextJS 前端

```bash
cd frontend/nextjs

# 安装依赖
npm install

# 开发模式
npm run dev

# 构建
npm run build

# 生产模式
npm run start
```

### A.3.3 前端配置

```bash
# frontend/nextjs/.env.local
NEXT_PUBLIC_GPTR_API_URL=http://localhost:8000
NEXT_PUBLIC_GA_MEASUREMENT_ID=xxx  # Google Analytics (可选)
```

---

## A.4 CLI 使用

### A.4.1 基本用法

```bash
# 基础研究报告
python cli.py "AI Agent 2024年最新进展" --report_type research_report

# 详细报告
python cli.py "量子计算最新突破" --report_type detailed_report

# 深度研究
python cli.py "气候变化对农业的影响" --report_type deep

# 指定语气
python cli.py "区块链技术" --report_type research_report --tone Formal

# 限制域名
python cli.py "机器学习" --report_type research_report --query_domains arxiv.org,ieee.org
```

### A.4.2 CLI 参数

| 参数 | 说明 | 示例 |
|------|------|------|
| `query` | 研究查询（位置参数） | `"AI Agent"` |
| `--report_type` | 报告类型 | `research_report` |
| `--tone` | 文章语气 | `Objective` |
| `--query_domains` | 域名限制 | `arxiv.org,ieee.org` |

---

## A.5 Python API 使用

### A.5.1 基础用法

```python
import asyncio
from gpt_researcher import GPTResearcher

async def main():
    # 创建研究者
    researcher = GPTResearcher(
        query="AI Agent 2024年最新进展",
        report_type="research_report",
        tone="Objective",
    )
    
    # 执行研究
    context = await researcher.conduct_research()
    
    # 撰写报告
    report = await researcher.write_report()
    
    print(report)
    print(f"Cost: ${researcher.get_costs():.4f}")

asyncio.run(main())
```

### A.5.2 深度研究

```python
from gpt_researcher import GPTResearcher

async def main():
    researcher = GPTResearcher(
        query="量子计算最新突破",
        report_type="deep",
    )
    
    def on_progress(progress):
        print(f"Depth: {progress.current_depth}/{progress.total_depth}")
        print(f"Queries: {progress.completed_queries}/{progress.total_queries}")
    
    context = await researcher.conduct_research(on_progress=on_progress)
    report = await researcher.write_report()
    print(report)

asyncio.run(main())
```

### A.5.3 使用本地文档

```python
researcher = GPTResearcher(
    query="分析我的文档",
    report_type="research_report",
    report_source="local",
)
# 文档放在 ./my-docs/ 目录
```

### A.5.4 使用 MCP

```python
researcher = GPTResearcher(
    query="研究任务",
    report_type="research_report",
    mcp_configs=[{
        "name": "my_tool",
        "command": "python",
        "args": ["my_mcp_server.py"],
    }],
    mcp_strategy="fast",
)
```

---

## A.6 调试

### A.6.1 日志配置

```bash
# 设置日志级别
LOGGING_LEVEL=DEBUG python main.py

# 或在 .env 中
LOGGING_LEVEL=DEBUG
```

### A.6.2 研究日志

```bash
# 查看研究日志
ls -la logs/
cat logs/research_20240101_120000.log
cat logs/research_20240101_120000.json
```

### A.6.3 LangSmith 追踪

```bash
# 启用 LangSmith
export LANGCHAIN_TRACING_V2=true
export LANGCHAIN_API_KEY=xxx
```

### A.6.4 VS Code 调试

```json
// .vscode/launch.json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Python: GPT Researcher",
            "type": "debugpy",
            "request": "launch",
            "program": "${workspaceFolder}/main.py",
            "console": "integratedTerminal",
            "env": {
                "OPENAI_API_KEY": "sk-xxx",
                "TAVILY_API_KEY": "tvly-xxx"
            }
        }
    ]
}
```

---

## A.7 测试

### A.7.1 运行测试

```bash
# 运行所有测试
pytest tests/ -v

# 运行特定测试
pytest tests/test_agent_discovery.py -v

# 运行匹配模式
pytest tests/ -k "test_research" -v

# 并行测试
pytest tests/ -n auto

# 生成覆盖率报告
pytest tests/ --cov=gpt_researcher --cov-report=html
```

### A.7.2 测试配置

```toml
# pyproject.toml
[tool.pytest.ini_options]
asyncio_mode = "strict"
addopts = "-v"
testpaths = ["tests"]
python_files = "test_*.py"
asyncio_fixture_loop_scope = "function"
```

### A.7.3 测试文件分类

| 类别 | 文件数 | 说明 |
|------|--------|------|
| 核心功能 | 20+ | Agent、Skills、Actions |
| Retriever | 15+ | 各检索器单元测试 |
| Scraper | 5+ | 爬虫功能测试 |
| API | 5+ | 后端 API 测试 |
| 集成 | 10+ | 端到端测试 |

### A.7.4 编写测试

```python
import pytest
from gpt_researcher import GPTResearcher

@pytest.mark.asyncio
async def test_basic_research():
    """测试基础研究功能"""
    researcher = GPTResearcher(
        query="Python 3.12 新特性",
        report_type="research_report",
        verbose=False,
    )
    
    context = await researcher.conduct_research()
    assert len(context) > 0
    
    report = await researcher.write_report()
    assert len(report) > 100
    assert "Python" in report
```

---

## A.8 贡献指南

### A.8.1 开发流程

```bash
# 1. Fork 仓库
# 2. 克隆 Fork
git clone https://github.com/YOUR_USERNAME/gpt-researcher.git

# 3. 创建分支
git checkout -b feature/my-feature

# 4. 开发并提交
git add .
git commit -m "feat: add new feature"

# 5. 推送
git push origin feature/my-feature

# 6. 创建 Pull Request
```

### A.8.2 代码规范

```bash
# 格式化
black gpt_researcher/
isort gpt_researcher/

# Lint
flake8 gpt_researcher/
mypy gpt_researcher/

# 类型检查
mypy gpt_researcher/ --ignore-missing-imports
```

### A.8.3 提交规范

```
feat: 新功能
fix: 修复
docs: 文档
style: 格式
refactor: 重构
test: 测试
chore: 构建/工具
```

### A.8.4 添加新检索器

```python
# gpt_researcher/retrievers/my_retriever/my_retriever.py
class MyRetriever:
    def __init__(self, query, headers=None, query_domains=None):
        self.query = query
        self.headers = headers or {}
    
    def search(self, max_results=10):
        # 实现搜索逻辑
        return [
            {"href": "url", "body": "content"},
            ...
        ]
```

```python
# gpt_researcher/retrievers/__init__.py
from .my_retriever.my_retriever import MyRetriever
```

```python
# gpt_researcher/actions/retriever.py
case "my_retriever":
    from gpt_researcher.retrievers import MyRetriever
    return MyRetriever
```

---

## A.9 常见问题

### A.9.1 API Key 问题

```bash
# 问题: 401 Unauthorized
# 解决: 检查 API Key 是否正确设置
echo $OPENAI_API_KEY
echo $TAVILY_API_KEY
```

### A.9.2 依赖问题

```bash
# 问题: ImportError
# 解决: 重新安装依赖
pip install -r requirements.txt --upgrade
```

### A.9.3 内存问题

```bash
# 问题: OOM
# 解决: 减少并发数
export MAX_SCRAPER_WORKERS=5
```

### A.9.4 网络问题

```bash
# 问题: Timeout
# 解决: 使用代理
export HTTP_PROXY=http://proxy:port
export HTTPS_PROXY=http://proxy:port
```

---

## A.10 术语表

| 术语 | 说明 |
|------|------|
| Agent | 研究智能体，编排研究流程 |
| Retriever | 检索后端，执行搜索 |
| Scraper | 爬虫，抓取网页内容 |
| Skill | 技能模块，封装子能力 |
| Action | 原子操作，最小功能单元 |
| LLM | 大语言模型 |
| MCP | Model Context Protocol |
| PromptFamily | 提示词族，管理 LLM 提示词 |
| VectorStore | 向量存储，文档嵌入 |
| ReportStore | 报告存储，JSON 持久化 |

---

> **下一节**: → `10-appendix-b.md` — 附录B 架构决策记录 (ADR)

---

☕️ 制作不易，请我喝咖啡☕️关注我➕