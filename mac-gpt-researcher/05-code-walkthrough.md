# 第5章 核心代码讲解（上）— Agent + Config + Actions

> **文件**: `docs/wangbin/05-code-walkthrough.md`  
> **预计 Token**: ~25,000  
> **核心内容**: GPTResearcher、Config、Actions 逐函数走读

---

## 5.1 Agent 模块走读 (gpt_researcher/agent.py)

### 5.1.1 模块概述

**文件**: `gpt_researcher/agent.py`  
**行数**: 794 行  
**核心类**: `GPTResearcher`  
**职责**: 主编排器，协调所有研究子组件

### 5.1.2 导入分析

```python
from .actions import (
    add_references, choose_agent, extract_headers,
    extract_sections, get_retrievers, get_search_results, table_of_contents,
)
from .config import Config
from .llm_provider import GenericLLMProvider
from .memory import Memory
from .prompts import get_prompt_family
from .skills.browser import BrowserManager
from .skills.context_manager import ContextManager
from .skills.curator import SourceCurator
from .skills.deep_research import DeepResearchSkill
from .skills.image_generator import ImageGenerator
from .skills.researcher import ResearchConductor
from .skills.writer import ReportGenerator
from .utils.enum import ReportSource, ReportType, Tone
from .utils.llm import create_chat_completion
from .vector_store import VectorStoreWrapper
```

**导入结构**:
- `actions`: 原子操作函数
- `skills`: 技能组件类
- `config/llm_provider/memory/prompts/vector_store`: 基础设施
- `utils.enum/utils.llm`: 工具函数

### 5.1.3 GPTResearcher 类详解

#### 5.1.3.1 `__init__` 方法

```python
def __init__(
    self,
    query: str,
    report_type: str = ReportType.ResearchReport.value,
    report_format: str = "markdown",
    report_source: str = ReportSource.Web.value,
    tone: Tone = Tone.Objective,
    source_urls: list[str] | None = None,
    document_urls: list[str] | None = None,
    complement_source_urls: bool = False,
    query_domains: list[str] | None = None,
    documents=None,
    vector_store=None,
    vector_store_filter=None,
    config_path=None,
    websocket=None,
    agent=None,
    role=None,
    parent_query: str = "",
    subtopics: list | None = None,
    visited_urls: set | None = None,
    verbose: bool = True,
    context=None,
    headers: dict | None = None,
    max_subtopics: int = 5,
    log_handler=None,
    prompt_family: str | None = None,
    mcp_configs: list[dict] | None = None,
    mcp_max_iterations: int | None = None,
    mcp_strategy: str | None = None,
    **kwargs
):
```

**参数分析**:

| 参数 | 类型 | 默认值 | 用途 |
|------|------|--------|------|
| `query` | str | 必填 | 研究查询 |
| `report_type` | str | research_report | 报告类型枚举值 |
| `report_format` | str | markdown | 输出格式 |
| `report_source` | str | web | 数据来源 |
| `tone` | Tone | Objective | 文章语气 |
| `source_urls` | list | None | 指定来源 URL |
| `document_urls` | list | None | 在线文档 URL |
| `query_domains` | list | None | 域名限制 |
| `config_path` | str | None | 配置文件路径 |
| `websocket` | WebSocket | None | WebSocket 连接 |
| `agent` | str | None | 预定义 Agent 名称 |
| `role` | str | None | 预定义 Agent 角色 |
| `parent_query` | str | "" | 父查询（子话题用） |
| `visited_urls` | set | None | 已访问 URL（去重用） |
| `mcp_configs` | list | None | MCP 服务器配置 |
| `mcp_strategy` | str | None | MCP 执行策略 |
| `**kwargs` | dict | - | 额外参数透传 |

**初始化逻辑**:

1. **配置加载**: `self.cfg = Config(config_path)`
2. **Retriever 初始化**: `self.retrievers = get_retrievers(self.headers, self.cfg)`
3. **记忆初始化**: `self.memory = Memory(...)`
4. **MCP 配置处理**: `self._process_mcp_configs(mcp_configs)`
5. **提示词族**: `self.prompt_family = get_prompt_family(...)`
6. **Skill 组件创建**:
   - `self.research_conductor = ResearchConductor(self)`
   - `self.report_generator = ReportGenerator(self)`
   - `self.context_manager = ContextManager(self)`
   - `self.scraper_manager = BrowserManager(self)`
   - `self.source_curator = SourceCurator(self)`
   - `self.deep_researcher = DeepResearchSkill(self)` (条件性)
   - `self.image_generator = ImageGenerator(self)` (可选)
7. **MCP 策略解析**: `self.mcp_strategy = self._resolve_mcp_strategy(...)`

**设计模式**: 组合模式（Composition）— Agent 由多个 Skill 组合而成。

**潜在问题**:
- 参数过多（30+），建议使用配置对象或 Builder 模式
- `**kwargs` 透传可能导致参数污染

#### 5.1.3.2 `conduct_research` 方法

```python
async def conduct_research(self, on_progress=None):
```

**功能**: 执行研究主流程

**流程**:
1. 记录研究开始日志
2. 判断是否深度研究 → 是则调用 `_handle_deep_research`
3. 未定义 agent/role → 调用 `choose_agent()` 自动选择
4. 调用 `self.research_conductor.conduct_research()` 执行研究
5. 预生成图像（如果启用）
6. 返回上下文

**关键代码路径**:
```python
if self.report_type == ReportType.DeepResearch.value and self.deep_researcher:
    return await self._handle_desearch(on_progress)

if not (self.agent and self.role):
    self.agent, self.role = await choose_agent(...)

self.context = await self.research_conductor.conduct_research()

# 预生成图像
if self.image_generator and self.image_generator.is_enabled():
    self.available_images = await self.image_generator.plan_and_generate_images(...)
```

**异常处理**: 各子组件自行处理异常，Agent 层不捕获。

#### 5.1.3.3 `write_report` 方法

```python
async def write_report(
    self,
    existing_headers: list = [],
    relevant_written_contents: list = [],
    ext_context=None,
    custom_prompt="",
) -> str:
```

**功能**: 撰写研究报告

**参数**:
- `existing_headers`: 已存在的标题（避免重复）
- `relevant_written_contents`: 已写内容（提供上下文）
- `ext_context`: 外部上下文（替代内部上下文）
- `custom_prompt`: 自定义提示词

**流程**:
1. 记录日志
2. 调用 `self.report_generator.write_report(...)`
3. 返回报告文本

**委托模式**: Agent 不直接实现报告生成，委托给 ReportGenerator。

#### 5.1.3.4 `quick_search` 方法

```python
async def quick_search(
    self,
    query: str,
    query_domains: list[str] = None,
    aggregated_summary: bool = False,
    all_retrievers: bool = False,
) -> list[Any] | str:
```

**功能**: 快速搜索（不触发完整研究流程）

**参数**:
- `aggregated_summary`: 是否返回聚合摘要
- `all_retrievers`: 是否查询所有检索器

**实现**:
```python
if all_retrievers and len(self.retrievers) > 1:
    search_results = await self._search_all_retrievers(query, query_domains)
else:
    search_results = await get_search_results(query, self.retrievers[0], ...)

if aggregated_summary:
    # 格式化结果并调用 LLM 生成摘要
    summary = await create_chat_completion(...)
    return summary

return search_results
```

#### 5.1.3.5 成本追踪方法

```python
def add_costs(self, cost: float) -> None:
    if not isinstance(cost, (float, int)):
        raise ValueError("Cost must be an integer or float")
    self.research_costs += cost
    step = self._current_step
    self.step_costs[step] = self.step_costs.get(step, 0.0) + cost
```

**功能**: 记录 API 成本，按步骤分类

**步骤分类**:
- `general`: 默认
- `agent_selection`: Agent 选择
- `research`: 研究执行
- `deep_research`: 深度研究
- `report_writing`: 报告撰写

#### 5.1.3.6 MCP 策略解析

```python
def _resolve_mcp_strategy(self, mcp_strategy: str | None, mcp_max_iterations: int | None) -> str:
```

**优先级**:
1. `mcp_strategy` 参数（新方式）
2. `mcp_max_iterations` 参数（向后兼容）
3. 配置 `MCP_STRATEGY`
4. 默认 `"fast"`

**向后兼容**:
- `mcp_max_iterations=0` → `"disabled"`
- `mcp_max_iterations=1` → `"fast"`
- `mcp_max_iterations=-1` → `"deep"`

---

## 5.2 Config 模块走读 (gpt_researcher/config/)

### 5.2.1 模块概述

**文件**:
- `config/config.py` — Config 类
- `config/variables/base.py` — BaseConfig TypedDict
- `config/variables/default.py` — DEFAULT_CONFIG 默认值

**职责**: 配置加载、解析、管理

### 5.2.2 BaseConfig (TypedDict)

```python
class BaseConfig(TypedDict):
    RETRIEVER: str
    EMBEDDING: str
    SIMILARITY_THRESHOLD: float
    FAST_LLM: str
    SMART_LLM: str
    STRATEGIC_LLM: str
    FAST_TOKEN_LIMIT: int
    SMART_TOKEN_LIMIT: int
    STRATEGIC_TOKEN_LIMIT: int
    # ... 30+ 字段
```

**设计**: 使用 TypedDict 提供类型安全，同时保持字典的灵活性。

### 5.2.3 DEFAULT_CONFIG

```python
DEFAULT_CONFIG: BaseConfig = {
    "RETRIEVER": "tavily",
    "EMBEDDING": "openai:text-embedding-3-small",
    "FAST_LLM": "openai:gpt-5.4-mini",
    "SMART_LLM": "openai:gpt-5.4",
    "STRATEGIC_LLM": "openai:gpt-5.4",
    "FAST_TOKEN_LIMIT": 6000,
    "SMART_TOKEN_LIMIT": 12000,
    "STRATEGIC_TOKEN_LIMIT": 8000,
    "TEMPERATURE": 0.4,
    "MAX_ITERATIONS": 3,
    "MAX_SUBTOPICS": 3,
    # ... 更多默认值
}
```

### 5.2.4 Config 类详解

#### 5.2.4.1 `__init__` 方法

```python
def __init__(self, config_path: str | None = None):
    self.config_path = config_path
    self.llm_kwargs: Dict[str, Any] = {}
    self.embedding_kwargs: Dict[str, Any] = {}

    config_to_use = self.load_config(config_path)
    self._set_attributes(config_to_use)
    self._set_embedding_attributes()
    self._set_llm_attributes()
    self._handle_deprecated_attributes()
    if config_to_use['REPORT_SOURCE'] != 'web':
        self._set_doc_path(config_to_use)
```

**初始化流程**:
1. 加载配置文件
2. 设置属性（环境变量覆盖）
3. 解析嵌入配置
4. 解析 LLM 配置
5. 处理弃用属性
6. 设置文档路径（非 Web 来源）

#### 5.2.4.2 `load_config` 方法

```python
def load_config(self, config_path: str | None) -> Dict[str, Any]:
    config = DEFAULT_CONFIG.copy()
    
    if config_path and os.path.exists(config_path):
        with open(config_path, 'r') as f:
            file_config = json.load(f)
        config.update(file_config)
    
    return config
```

**加载策略**:
1. 从 DEFAULT_CONFIG 复制
2. 如果配置文件存在，读取并合并
3. 返回最终配置

#### 5.2.4.3 `_set_attributes` 方法

```python
def _set_attributes(self, config: Dict[str, Any]) -> None:
    for key, value in config.items():
        env_value = os.getenv(key)
        if env_value is not None:
            value = self.convert_env_value(key, env_value, BaseConfig.__annotations__[key])
        setattr(self, key.lower(), value)
```

**环境变量覆盖逻辑**:
- 遍历所有配置项
- 检查同名环境变量
- 如果存在，转换类型并覆盖
- 设置为实例属性（小写）

#### 5.2.4.4 `parse_retrievers` 方法

```python
def parse_retrievers(self, retriever_env: str) -> list[str]:
    return [r.strip() for r in retriever_env.split(",") if r.strip()]
```

**功能**: 解析逗号分隔的检索器字符串

**示例**: `"tavily,exa,duckduckgo"` → `["tavily", "exa", "duckduckgo"]`

#### 5.2.4.5 `parse_llm` 方法

```python
def parse_llm(self, llm: str) -> tuple[str, str]:
    parts = llm.split(":", 1)
    if len(parts) == 2:
        return parts[0], parts[1]
    return parts[0], parts[0]
```

**功能**: 解析 `"provider:model"` 格式

**示例**: `"openai:gpt-5.4"` → `("openai", "gpt-5.4")`

#### 5.2.4.6 `parse_embedding` 方法

```python
def parse_embedding(self, embedding: str) -> tuple[str, str]:
    return self.parse_llm(embedding)  # 格式相同
```

#### 5.2.4.7 `_handle_deprecated_attributes` 方法

```python
def _handle_deprecated_attributes(self) -> None:
    if os.getenv("EMBEDDING_PROVIDER") is not None:
        warnings.warn("EMBEDDING_PROVIDER is deprecated...", FutureWarning, stacklevel=2)
        self.embedding_provider = os.environ["EMBEDDING_PROVIDER"] or self.embedding_provider
    # ... 更多弃用处理
```

**弃用项**:
- `EMBEDDING_PROVIDER` → `EMBEDDING`
- `LLM_PROVIDER` → `FAST_LLM`/`SMART_LLM`
- `FAST_LLM_MODEL` → `FAST_LLM`
- `SMART_LLM_MODEL` → `SMART_LLM`

**向后兼容**: 通过 `warnings.warn()` 提示用户迁移。

---

## 5.3 Actions 模块走读 (gpt_researcher/actions/)

### 5.3.1 模块概述

**目录**: `gpt_researcher/actions/`  
**文件数**: 7 个模块  
**职责**: 原子操作，被 Skills 调用

### 5.3.2 query_processing.py

#### 5.3.2.1 `get_search_results` 函数

```python
async def get_search_results(
    query: str,
    retriever: Any,
    query_domains: List[str] = None,
    researcher=None,
    max_results: int | None = None,
) -> List[Dict[str, Any]]:
```

**功能**: 获取搜索结果

**关键逻辑**:
```python
# 检查是否为 MCP 检索器
if "mcpretriever" in retriever.__name__.lower():
    search_retriever = retriever(query, query_domains=query_domains, researcher=researcher)
else:
    search_retriever = retriever(query, query_domains=query_domains)

# 阻塞调用卸载到线程
return await asyncio.to_thread(search_retriever.search, **search_kwargs)
```

**设计决策**:
- MCP 检索器需要 `researcher` 实例（访问 mcp_configs 和 cfg）
- 使用 `asyncio.to_thread()` 避免阻塞事件循环

#### 5.3.2.2 `generate_sub_queries` 函数

```python
async def generate_sub_queries(
    query: str,
    parent_query: str,
    report_type: str,
    context: List[Dict[str, Any]],
    cfg: Config,
    cost_callback: callable = None,
    prompt_family: type[PromptFamily] | PromptFamily = PromptFamily,
    **kwargs
) -> List[str]:
```

**功能**: 使用 Strategic LLM 生成子查询

**降级策略**:
```
Strategic LLM (无 max_tokens)
  → 失败重试 (max_tokens=config)
    → 降级到 Smart LLM
```

**响应解析**:
```python
return _normalize_sub_queries(json_repair.loads(response), query)
```

**`_normalize_sub_queries`**: 防御性解析，处理各种 LLM 输出格式：
- `{"queries": [...]}` → 提取列表
- `{"query": "..."}` → 单元素列表
- `"bare string"` → 单元素列表
- `None` → 回退到原始查询

#### 5.3.2.3 `plan_research_outline` 函数

```python
async def plan_research_outline(
    query: str,
    search_results: List[Dict[str, Any]],
    agent_role_prompt: str,
    cfg: Config,
    parent_query: str,
    report_type: str,
    cost_callback: callable = None,
    retriever_names: List[str] = None,
    **kwargs
) -> List[str]:
```

**功能**: 规划研究大纲（生成子查询）

**MCP 优化**:
```python
if retriever_names and ("mcp" in retriever_names or "MCPRetriever" in retriever_names):
    mcp_only = (len(retriever_names) == 1 and 
               ("mcp" in retriever_names or "MCPRetriever" in retriever_names))
    if mcp_only:
        return [query]  # MCP 唯一检索器时跳过子查询生成
```

**理由**: MCP 工具本身具有智能搜索能力，不需要额外的子查询分解。

### 5.3.3 report_generation.py

#### 5.3.3.1 `generate_report` 函数

```python
async def generate_report(
    query: str,
    context: str,
    agent_role_prompt: str,
    config: Config,
    websocket=None,
    cost_callback: callable = None,
    prompt_family: type[PromptFamily] | PromptFamily = PromptFamily,
    **kwargs
) -> str:
```

**功能**: 生成最终报告

**流程**:
1. 构建报告提示词
2. 调用 `create_chat_completion()` 获取 Smart LLM 响应
3. 流式传输到 WebSocket
4. 返回报告文本

#### 5.3.3.2 `write_report_introduction` 函数

```python
async def write_report_introduction(
    query: str,
    context: str,
    agent_role_prompt: str,
    config: Config,
    websocket=None,
    cost_callback: callable = None,
    prompt_family: type[PromptFamily] | PromptFamily = PromptFamily,
    **kwargs
) -> str:
```

**功能**: 生成引言部分

**特点**:
- 使用较低温度 (0.25) 保证一致性
- 流式传输到 WebSocket
- 错误时返回空字符串

#### 5.3.3.3 `write_conclusion` 函数

```python
async def write_conclusion(
    query: str,
    context: str,
    agent_role_prompt: str,
    config: Config,
    websocket=None,
    cost_callback: callable = None,
    prompt_family: type[PromptFamily] | PromptFamily = PromptFamily,
    **kwargs
) -> str:
```

**功能**: 生成结论部分

#### 5.3.3.4 `generate_draft_section_titles` 函数

```python
async def generate_draft_section_titles(
    query: str,
    current_subtopic: str,
    context: str,
    role: str,
    config: Config,
    websocket=None,
    cost_callback: callable = None,
    prompt_family: type[PromptFamily] | PromptFamily = PromptFamily,
    **kwargs
) -> List[str]:
```

**功能**: 为子话题生成草稿标题

### 5.3.4 agent_creator.py

#### 5.3.4.1 `choose_agent` 函数

```python
async def choose_agent(
    query,
    cfg,
    parent_query=None,
    cost_callback: callable = None,
    headers=None,
    prompt_family: type[PromptFamily] | PromptFamily = PromptFamily,
    **kwargs
):
```

**功能**: 自动选择 Agent 角色

**流程**:
1. 构建查询（含父查询上下文）
2. 调用 Smart LLM
3. 解析 JSON 响应
4. 返回 `(agent_name, agent_role_prompt)`

**提示词**: `prompt_family.auto_agent_instructions()`

**响应格式**:
```json
{
    "server": "Research Agent",
    "agent_role_prompt": "You are an AI critical thinker..."
}
```

#### 5.3.4.2 `handle_json_error` 函数

```python
async def handle_json_error(response: str | None):
```

**功能**: JSON 解析失败的容错处理

**回退策略**:
1. `json_repair.loads()` — 修复常见 JSON 错误
2. `extract_json_with_regex()` — 正则提取 JSON 对象
3. 默认 Agent — `"Default Agent"` + 通用角色提示

#### 5.3.4.3 `extract_json_with_regex` 函数

```python
def extract_json_with_regex(response: str | None) -> str | None:
    json_match = re.search(r"{.*}", response, re.DOTALL)
    if json_match:
        return json_match.group(0)
    return None
```

**注意**: 使用贪婪匹配 `.*` 捕获完整对象（包括嵌套 `}`）。

### 5.3.5 retriever.py

#### 5.3.5.1 `get_retriever` 函数

```python
def get_retriever(retriever: str):
```

**功能**: 根据名称获取检索器类

**实现**: Python 3.10+ match-case 模式匹配

```python
match retriever:
    case "google":
        from gpt_researcher.retrievers import GoogleSearch
        return GoogleSearch
    case "tavily":
        from gpt_researcher.retrievers import TavilySearch
        return TavilySearch
    # ... 20+ case
    case _:
        return None
```

**设计**: 懒导入 — 仅在需要时加载对应模块。

#### 5.3.5.2 `get_retrievers` 函数

```python
def get_retrievers(headers: dict[str, str], cfg):
```

**功能**: 确定使用的检索器列表

**优先级**:
1. `headers.get("retrievers")` — 多检索器
2. `headers.get("retriever")` — 单检索器
3. `cfg.retrievers` — 配置多检索器
4. `cfg.retriever` — 配置单检索器
5. 默认 — `TavilySearch`

**回退机制**: 无效检索器名称使用默认检索器替代。

```python
retriever_classes = [get_retriever(r) or get_default_retriever() for r in retrievers]
```

### 5.3.6 web_scraping.py

#### 5.3.6.1 `scrape_urls` 函数

```python
async def scrape_urls(
    urls, cfg: Config, worker_pool: WorkerPool
) -> tuple[list[dict[str, Any]], list[dict[str, Any]]]:
```

**功能**: 抓取 URL 列表

**流程**:
1. 创建 Scraper 实例
2. 调用 `scraper.run()` 并行抓取
3. 提取图像 URL
4. 返回 `(scraped_data, images)`

**资源清理**:
```python
finally:
    if scraper is not None and getattr(scraper, "session", None) is not None:
        scraper.session.close()
```

### 5.3.7 markdown_processing.py

#### 5.3.7.1 `extract_headers` 函数

```python
def extract_headers(markdown_text: str) -> list[dict]:
```

**功能**: 从 Markdown 提取标题

**实现**: 正则匹配 `#{1,6} 标题`

#### 5.3.7.2 `extract_sections` 函数

```python
def extract_sections(markdown_text: str) -> list[dict]:
```

**功能**: 从 Markdown 提取章节

#### 5.3.7.3 `add_references` 函数

```python
def add_references(report_markdown: str, visited_urls: set) -> str:
```

**功能**: 为报告添加参考文献章节

### 5.3.8 utils.py (actions)

#### 5.3.8.1 `stream_output` 函数

```python
async def stream_output(
    type, content, output, websocket=None, output_log=True, metadata=None
):
```

**功能**: 流式输出到 WebSocket

**行为**:
- 如果有 WebSocket，发送 JSON
- 如果 `output_log=True`，同时记录到日志

#### 5.3.8.2 `calculate_cost` 函数

```python
def calculate_cost(prompt_tokens: int, completion_tokens: int, model: str) -> float:
```

**功能**: 计算 API 调用成本

**注意**: 这是一个简化版本，更精确的计算在 `utils/costs.py` 中。

---

## 5.4 代码质量分析

### 5.4.1 优点

1. **类型注解**: 全代码使用 Python 类型提示
2. **文档字符串**: 所有公共函数含 docstring
3. **错误处理**: 多层回退机制（LLM 降级、JSON 解析回退）
4. **异步优先**: 全异步 I/O 设计
5. **配置灵活**: 环境变量 + 配置文件 + 默认值

### 5.4.2 潜在问题

| 问题 | 位置 | 风险 | 建议 |
|------|------|------|------|
| Agent 类过大 | agent.py | 高 | 拆分为多个类 |
| 参数透传 `**kwargs` | agent.py | 中 | 使用配置对象 |
| 全局状态修改 | config.py | 中 | 使用上下文管理器 |
| 硬编码模型列表 | llm_provider/base.py | 低 | 动态发现 |
| 正则提取 JSON | agent_creator.py | 中 | 使用更健壮的解析 |

### 5.4.3 性能瓶颈

| 瓶颈 | 影响 | 优化建议 |
|------|------|---------|
| LLM 调用延迟 | 高 | 缓存、批量请求 |
| 网页抓取 | 中 | 增加并发、连接池 |
| 嵌入计算 | 中 | 批量嵌入 |
| JSON 解析 | 低 | 使用 orjson |

### 5.4.4 设计模式应用

| 模式 | 应用 | 文件 |
|------|------|------|
| 工厂模式 | `get_retriever()`, `GenericLLMProvider.from_provider()` | retriever.py, base.py |
| 策略模式 | Retriever/Scraper 实现 | retrievers/*, scraper/* |
| 模板方法 | Skills 继承体系 | skills/* |
| 装饰器 | `@tool` 装饰器 | utils/tools.py |
| 观察者 | WebSocket 流式传输 | actions/utils.py |
| 外观模式 | `GPTResearcher` | agent.py |
| 依赖注入 | Skills 接收 researcher | skills/* |

---

## 5.5 关键设计决策总结

| 决策 | 选择 | 理由 |
|------|------|------|
| 异步框架 | asyncio | I/O 密集型任务 |
| LLM 降级 | Strategic → Smart → Fast | 可靠性优先 |
| JSON 解析 | json_repair → regex → default | 容错性优先 |
| 配置优先级 | 环境变量 > 文件 > 默认 | 12-Factor App |
| 插件注册 | match-case 工厂 | 简洁可扩展 |
| 并发控制 | Semaphore + RateLimiter | 资源保护 |

---

> **下一节**: → `05-code-walkthrough-b.md` — 核心代码讲解（下）Skills + Retrievers + Scraper

---

☕️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../res/promotion.jpg)