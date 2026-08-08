# 第5章 核心代码讲解（下）— Skills + Retrievers + Scraper

> **文件**: `docs/wangbin/05-code-walkthrough-b.md`  
> **预计 Token**: ~25,000  
> **核心内容**: Skills、Retrievers、Scraper、LLM Provider、MCP 逐函数走读

---

## 5.6 Skills 模块走读 (gpt_researcher/skills/)

### 5.6.1 模块概述

**目录**: `gpt_researcher/skills/`  
**文件数**: 7 个模块  
**职责**: 封装研究子能力，被 Agent 调用

### 5.6.2 researcher.py (ResearchConductor)

#### 5.6.2.1 类概述

```python
class ResearchConductor:
    """管理和协调研究过程"""
    
    def __init__(self, researcher):
        self.researcher = researcher
        self.logger = logging.getLogger('research')
        self.json_handler = get_json_handler()
        self._mcp_results_cache = None
        self._mcp_cache_lock = asyncio.Lock()
        self._mcp_query_count = 0
```

**关键属性**:
- `_mcp_results_cache`: MCP 结果缓存
- `_mcp_cache_lock`: 保护缓存的并发访问
- `_mcp_query_count`: MCP 查询计数

#### 5.6.2.2 `conduct_research` 方法

```python
async def conduct_research(self):
```

**功能**: 执行研究主流程

**流程**:
1. Agent 选择（如果未定义）
2. 判断 MCP 检索器
3. 根据数据来源类型执行不同策略:
   - `source_urls`: 使用指定 URL
   - `Web`: 联网搜索
   - `Local`: 本地文档
   - `Hybrid`: Web + 本地混合
   - `Azure`: Azure Blob
   - `LangChainDocuments`: LangChain 文档
4. 返回研究数据

**数据来源路由**:
```python
if self.researcher.source_urls:
    research_data = await self._get_context_by_urls(self.researcher.source_urls)
elif self.researcher.report_source == ReportSource.Web.value:
    research_data = await self._get_context_by_web_search(...)
elif self.researcher.report_source == ReportSource.Local.value:
    document_data = await DocumentLoader(...).load()
    research_data = await self._get_context_by_web_search(...)
elif self.researcher.report_source == ReportSource.Hybrid.value:
    # 并发执行 Web + 本地
    docs_context, web_context = await asyncio.gather(...)
    research_data = join_local_web_documents(docs_context, web_context)
```

#### 5.6.2.3 `_get_context_by_web_search` 方法

```python
async def _get_context_by_web_search(self, query, documents, query_domains):
```

**功能**: 通过联网搜索获取上下文

**流程**:
1. 规划研究（生成子查询）
2. 对每个子查询执行搜索
3. 抓取搜索结果 URL
4. 摘要抓取内容
5. 累积到上下文

#### 5.6.2.4 `plan_research` 方法

```python
async def plan_research(self, query, query_domains=None):
```

**功能**: 规划研究策略

**流程**:
1. 获取初始搜索结果
2. 调用 LLM 生成研究大纲
3. 返回子查询列表

### 5.6.3 writer.py (ReportGenerator)

#### 5.6.3.1 类概述

```python
class ReportGenerator:
    """基于研究数据生成报告"""
    
    def __init__(self, researcher):
        self.researcher = researcher
        self.research_params = {
            "query": self.researcher.query,
            "agent_role_prompt": ...,
            "report_type": ...,
            "report_source": ...,
            "tone": ...,
            "websocket": ...,
            "cfg": ...,
            "headers": ...,
        }
```

#### 5.6.3.2 `write_report` 方法

```python
async def write_report(
    self,
    existing_headers: list = [],
    relevant_written_contents: list = [],
    ext_context=None,
    custom_prompt="",
    available_images: list = None,
) -> str:
```

**功能**: 撰写报告

**空内容保护**:
```python
_ctx = "\n".join(context) if isinstance(context, list) else str(context or "")
if not _ctx.strip():
    return (
        f'I could not gather any source material for "{self.researcher.query}". '
        "No sources were retrieved..."
    )
```

**设计亮点**: 避免在无任何来源时生成伪造报告。

#### 5.6.3.3 `write_report_conclusion` 方法

```python
async def write_report_conclusion(self, report_content: str) -> str:
```

**功能**: 生成结论

#### 5.6.3.4 `write_introduction` 方法

```python
async def write_introduction(self):
```

**功能**: 生成引言

#### 5.6.3.5 `get_subtopics` 方法

```python
async def get_subtopics(self):
```

**功能**: 生成子话题列表

**实现**: 调用 `construct_subtopics()` 工具函数

### 5.6.4 browser.py (BrowserManager)

#### 5.6.4.1 类概述

```python
class BrowserManager:
    """管理网页抓取和内容提取"""
    
    def __init__(self, researcher):
        self.researcher = researcher
        self.worker_pool = WorkerPool(
            researcher.cfg.max_scraper_workers,
            researcher.cfg.scraper_rate_limit_delay
        )
```

#### 5.6.4.2 `browse_urls` 方法

```python
async def browse_urls(self, urls: list[str]) -> list[dict]:
```

**功能**: 抓取 URL 列表

**流程**:
1. 调用 `scrape_urls()` 抓取内容
2. 选择顶部图像
3. 添加到 researcher 的源和图像集合

#### 5.6.4.3 `select_top_images` 方法

```python
def select_top_images(self, images: list[dict], k: int = 2) -> list[str]:
```

**功能**: 选择最相关的图像并去重

**去重算法**:
```python
for img in sorted(images, key=lambda im: im["score"], reverse=True):
    img_hash = get_image_hash(img['url'])
    if img_hash and img_hash not in seen_hashes and img['url'] not in current_research_images:
        seen_hashes.add(img_hash)
        unique_images.append(img["url"])
        if len(unique_images) == k:
            break
```

**策略**: 按分数降序排列，选择前 k 个唯一图像。

### 5.6.5 context_manager.py (ContextManager)

#### 5.6.5.1 类概述

```python
class ContextManager:
    """管理研究上下文"""
    
    def __init__(self, researcher):
        self.researcher = researcher
```

#### 5.6.5.2 `get_similar_written_contents_by_draft_section_titles` 方法

```python
async def get_similar_written_contents_by_draft_section_titles(
    self,
    current_subtopic: str,
    draft_section_titles: list[str],
    written_contents: list[dict],
    max_results: int = 10
) -> list[str]:
```

**功能**: 基于章节标题查找相似的已写内容

**用途**: 避免子话题报告中的内容重复

### 5.6.6 curator.py (SourceCurator)

#### 5.6.6.1 类概述

```python
class SourceCurator:
    """基于相关性、可信度和可靠性评估和排序研究来源"""
    
    def __init__(self, researcher):
        self.researcher = researcher
```

#### 5.6.6.2 `curate_sources` 方法

```python
async def curate_sources(
    self,
    source_data: List,
    max_results: int = 10,
) -> List:
```

**功能**: 审查和排序来源

**流程**:
1. 调用 LLM 评估来源
2. 解析 JSON 响应
3. 返回排序后的来源列表

**容错**: 失败时返回原始来源数据

### 5.6.7 deep_research.py (DeepResearchSkill)

#### 5.6.7.1 模块概述

**文件**: `gpt_researcher/skills/deep_research.py`  
**行数**: 646 行  
**核心类**: `DeepResearchSkill`  
**职责**: 递归深度研究

#### 5.6.7.2 辅助解析函数

**`parse_search_queries_response`**:
```python
def parse_search_queries_response(response: str, num_queries: int) -> List[Dict[str, str]]:
```

**功能**: 解析 LLM 返回的搜索查询

**支持格式**:
- JSON 数组: `[{"query": "...", "researchGoal": "..."}]`
- JSON 对象: `{"queries": [...]}`
- 纯文本: `Query: ...` / `Goal: ...`

**容错**: `json_repair` → 正则 → 回退

**`parse_follow_up_questions_response`**:
```python
def parse_follow_up_questions_response(response: str, num_questions: int) -> List[str]:
```

**功能**: 解析后续问题

**`parse_research_results_response`**:
```python
def parse_research_results_response(response: str, num_learnings: int) -> Dict[str, Any]:
```

**功能**: 解析研究结果为 `learnings` 和 `followUpQuestions`

**返回格式**:
```python
{
    "learnings": ["insight1", "insight2"],
    "followUpQuestions": ["question1", "question2"],
    "citations": {"insight1": "url1"}
}
```

#### 5.6.7.3 JSON 提取辅助

```python
JSON_BLOCK_PATTERNS = [
    re.compile(r"```(?:json)?\s*(?P<payload>[\s\S]*?)```", re.IGNORECASE),
    re.compile(r"(?P<payload>\[[\s\S]*\])"),
    re.compile(r"(?P<payload>\{[\s\S]*\})"),
]

def _extract_json_payloads(response: str) -> list[str]:
    """从 LLM 响应中提取所有 JSON 候选"""
```

**策略**: 依次尝试 code block、数组、对象模式。

#### 5.6.7.4 DeepResearchSkill 类

```python
class DeepResearchSkill:
    def __init__(self, researcher):
        self.researcher = researcher
        self.breadth = researcher.cfg.deep_research_breadth
        self.depth = researcher.cfg.deep_research_depth
        self.concurrency_limit = researcher.cfg.deep_research_concurrency
```

**核心参数**:
- `breadth`: 每层查询数（默认 3）
- `depth`: 递归深度（默认 2）
- `concurrency`: 并发数（默认 4）

---

## 5.7 Retrievers 模块走读 (gpt_researcher/retrievers/)

### 5.7.1 模块概述

**目录**: `gpt_researcher/retrievers/`  
**子目录**: 20+ 个检索器实现  
**统一接口**: 所有检索器实现 `search()` 方法

### 5.7.2 Tavily 检索器

#### 5.7.2.1 类概述

```python
class TavilySearch:
    def __init__(self, query, headers=None, topic="general", query_domains=None):
        self.query = query
        self.headers = headers or {}
        self.topic = topic
        self.base_url = "https://api.tavily.com/search"
        self.api_key = self.get_api_key()
        self.query_domains = query_domains or None
```

#### 5.7.2.2 API Key 获取

```python
def get_api_key(self):
    api_key = self.headers.get("tavily_api_key")
    if not api_key:
        try:
            api_key = os.environ["TAVILY_API_KEY"]
        except KeyError:
            print("Tavily API key not found...")
            return ""
    return api_key
```

**优先级**: headers → 环境变量 → 空字符串

#### 5.7.2.3 `search` 方法

```python
def search(self, max_results=10):
```

**关键处理**:
1. 转换 Google `site:` 操作符为 Tavily `include_domains`
2. 截断查询至 400 字符（Tavily 限制）
3. 发送 POST 请求
4. 解析响应为统一格式

**响应格式**:
```python
[
    {"href": "url", "body": "content"},
    ...
]
```

**错误处理**: 失败时返回空列表。

### 5.7.3 MCP 检索器

#### 5.7.3.1 类概述

```python
class MCPRetriever:
    """基于 MCP 工具的智能检索器"""
    
    def __init__(self, query, headers=None, query_domains=None, websocket=None, researcher=None, **kwargs):
        self.query = query
        self.researcher = researcher
        self.mcp_configs = self._get_mcp_configs()
        self.cfg = self._get_config()
        self.client_manager = MCPClientManager(self.mcp_configs)
        self.tool_selector = MCPToolSelector(self.cfg, self.researcher)
        self.mcp_researcher = MCPResearchSkill(self.cfg, self.researcher)
        self.streamer = MCPStreamer(self.websocket)
```

#### 5.7.3.2 两阶段搜索

```python
async def search_async(self, max_results: int = 10) -> List[Dict[str, str]]:
    # 阶段 1: 获取工具列表
    all_tools = await self.client_manager.get_or_create_client()
    
    # 阶段 2: LLM 工具选择
    selected_tools = await self.tool_selector.select_tools(self.query, all_tools)
    
    # 阶段 3: 研究执行
    results = await self.mcp_researcher.execute_research(self.query, selected_tools)
    
    return results
```

### 5.7.4 其他检索器（简要）

| 检索器 | 文件 | 特点 |
|--------|------|------|
| DuckDuckGo | `duckduckgo/duckduckgo.py` | 免费，无需 API Key |
| Google | `google/google.py` | Custom Search JSON API |
| Bing | `bing/bing.py` | Bing Search API |
| Brave | `brave/brave.py` | Brave Search API |
| ArXiv | `arxiv/arxiv.py` | 学术论文 |
| Semantic Scholar | `semantic_scholar/semantic_scholar.py` | 学术搜索 |
| PubMed | `pubmed_central/pubmed_central.py` | 医学文献 |
| Custom | `custom/custom.py` | 用户实现 |

---

## 5.8 Scraper 模块走读 (gpt_researcher/scraper/)

### 5.8.1 模块概述

**目录**: `gpt_researcher/scraper/`  
**子目录**: 8 种爬虫实现  
**核心类**: `Scraper`

### 5.8.2 Scraper 类

#### 5.8.2.1 `__init__` 方法

```python
def __init__(self, urls, user_agent, scraper, worker_pool: WorkerPool):
    # URL 去重
    unique_urls = list(dict.fromkeys(urls))
    self.urls = unique_urls
    self.session = requests.Session()
    self.session.headers.update({"User-Agent": user_agent})
    self.scraper = scraper
    self.worker_pool = worker_pool
```

**URL 去重**: 使用 `dict.fromkeys()` 保持顺序并去重。

#### 5.8.2.2 `run` 方法

```python
async def run(self):
    contents = await asyncio.gather(
        *(self.extract_data_from_url(url, self.session) for url in self.urls)
    )
    res = [content for content in contents if content["raw_content"] is not None]
    return res
```

**并行抓取**: 使用 `asyncio.gather()` 并行处理所有 URL。

#### 5.8.2.3 `get_scraper` 方法

```python
def get_scraper(self, link):
    SCRAPER_CLASSES = {
        "pdf": PyMuPDFScraper,
        "arxiv": ArxivScraper,
        "bs": BeautifulSoupScraper,
        "web_base_loader": WebBaseLoaderScraper,
        "browser": BrowserScraper,
        "nodriver": NoDriverScraper,
        "tavily_extract": TavilyExtract,
        "firecrawl": FireCrawl,
    }
    
    path = urlparse(link).path
    if path.lower().endswith(".pdf"):
        scraper_key = "pdf"
    elif "arxiv.org" in link:
        scraper_key = "arxiv"
    else:
        scraper_key = self.scraper
    
    scraper_class = SCRAPER_CLASSES.get(scraper_key)
    if scraper_class is None:
        raise Exception("Scraper not found.")
    
    return scraper_class
```

**路由逻辑**:
1. `.pdf` → PyMuPDFScraper
2. `arxiv.org` → ArxivScraper
3. 其他 → 配置的默认爬虫

#### 5.8.2.4 `extract_data_from_url` 方法

```python
async def extract_data_from_url(self, link, session):
    async with self.worker_pool.throttle():
        try:
            Scraper = self.get_scraper(link)
            scraper = Scraper(link, session)
            
            if hasattr(scraper, "scrape_async"):
                content, image_urls, title = await scraper.scrape_async()
            else:
                content, image_urls, title = await asyncio.get_running_loop().run_in_executor(
                    self.worker_pool.executor, scraper.scrape
                )
            
            if len(content) < 100:
                return {"url": link, "raw_content": None, ...}
            
            return {"url": link, "raw_content": content, "image_urls": image_urls, "title": title}
        except Exception as e:
            return {"url": link, "raw_content": None, "image_urls": [], "title": ""}
```

**并发控制**: `worker_pool.throttle()` 同时控制并发数和限速。

**异步适配**: 优先使用 `scrape_async()`，否则通过线程池执行 `scrape()`。

**内容验证**: 内容少于 100 字符视为抓取失败。

### 5.8.3 BeautifulSoup 爬虫

```python
class BeautifulSoupScraper:
    def scrape(self):
        response = self.session.get(self.url, timeout=10)
        soup = BeautifulSoup(response.text, "html.parser")
        # 清理和提取文本
        ...
        return text, image_urls, title
```

### 5.8.4 Browser 爬虫 (Selenium)

```python
class BrowserScraper:
    def scrape(self):
        self.setup_driver()
        self._visit_google_and_save_cookies()
        self._load_saved_cookies()
        text, image_urls, title = self.scrape_text_with_selenium()
        return text, image_urls, title
```

**特点**:
- 支持 JS 渲染
- Cookie 管理
- 反爬绕过

### 5.8.5 PyMuPDF 爬虫

```python
class PyMuPDFScraper:
    def scrape(self):
        doc = fitz.open(self.url)
        text = ""
        for page in doc:
            text += page.get_text()
        return text, [], doc.metadata.get("title", "")
```

**用途**: PDF 文档解析

---

## 5.9 LLM Provider 模块走读 (gpt_researcher/llm_provider/)

### 5.9.1 模块概述

**目录**: `gpt_researcher/llm_provider/`  
**核心类**: `GenericLLMProvider`  
**职责**: 统一 27 家 LLM 提供商的调用接口

### 5.9.2 GenericLLMProvider 类

#### 5.9.2.1 `from_provider` 工厂方法

```python
@classmethod
def from_provider(cls, provider: str, chat_log: str = None, verbose: bool = True, **kwargs):
    if provider == "openai":
        from langchain_openai import ChatOpenAI
        kwargs.setdefault("stream_usage", True)
        llm = ChatOpenAI(**kwargs)
    elif provider == "anthropic":
        from langchain_anthropic import ChatAnthropic
        llm = ChatAnthropic(**kwargs)
    elif provider == "ollama":
        from langchain_ollama import ChatOllama
        llm = ChatOllama(base_url=os.environ["OLLAMA_BASE_URL"], **kwargs)
    # ... 24 更多提供商
    
    return cls(llm, chat_log, verbose)
```

**设计**: 每个提供商独立的导入和初始化逻辑。

#### 5.9.2.2 特殊模型列表

```python
NO_SUPPORT_TEMPERATURE_MODELS = [
    "deepseek/deepseek-reasoner",
    "o1-mini", "o1", "o3-mini", "o3", "o4-mini",
    "gpt-5", "gpt-5-mini", "gpt-5-nano",
    "gpt-5.4", "gpt-5.4-mini", "gpt-5.4-nano", "gpt-5.4-pro",
    "claude-sonnet-4-5", "claude-sonnet-4-6", "claude-opus-4-5", ...
]

SUPPORT_REASONING_EFFORT_MODELS = [
    "o3-mini", "o3", "o4-mini",
    "gpt-5.4", "gpt-5.4-mini", "gpt-5.4-nano", "gpt-5.4-pro",
    "gpt-5.5", "gpt-5.5-pro",
]
```

**用途**: 根据模型能力调整请求参数。

### 5.9.3 ChatLogger 类

```python
class ChatLogger:
    def __init__(self, fname: str):
        self.fname = fname
        self._lock = asyncio.Lock()
    
    async def log_request(self, messages, response):
        async with self._lock:
            async with aiofiles.open(self.fname, mode="a", encoding="utf-8") as handle:
                await handle.write(json.dumps({
                    "messages": messages,
                    "response": response,
                    "stacktrace": traceback.format_exc()
                }) + "\n")
```

**用途**: 记录所有 LLM 请求和响应，用于调试和审计。

---

## 5.10 MCP 模块走读 (gpt_researcher/mcp/)

### 5.10.1 模块概述

**目录**: `gpt_researcher/mcp/`  
**文件**: client.py, tool_selector.py, research.py, streaming.py  
**职责**: MCP 服务器连接、工具选择、研究执行

### 5.10.2 MCPClientManager 类

#### 5.10.2.1 `convert_configs_to_langchain_format` 方法

```python
def convert_configs_to_langchain_format(self) -> Dict[str, Dict[str, Any]]:
    server_configs = {}
    for i, config in enumerate(self.mcp_configs):
        server_name = config.get("name", f"mcp_server_{i+1}")
        server_config = {}
        
        connection_url = config.get("connection_url")
        if connection_url:
            if connection_url.startswith(("wss://", "ws://")):
                server_config["transport"] = "websocket"
                server_config["url"] = connection_url
            elif connection_url.startswith(("https://", "http://")):
                server_config["transport"] = "streamable_http"
                server_config["url"] = connection_url
        else:
            connection_type = config.get("connection_type", "stdio")
            server_config["transport"] = connection_type
            if server_config.get("transport") == "stdio":
                server_config["command"] = config.get("command")
                server_config["args"] = config.get("args", [])
        
        server_configs[server_name] = server_config
    return server_configs
```

**功能**: 将 GPT Researcher MCP 配置转换为 langchain-mcp-adapters 格式

#### 5.10.2.2 `get_or_create_client` 方法

```python
async def get_or_create_client(self) -> Optional[object]:
    async with self._client_lock:
        if self._client is not None:
            return self._client
        
        if not HAS_MCP_ADAPTERS:
            return None
        
        server_configs = self.convert_configs_to_langchain_format()
        self._client = MultiServerMCPClient(server_configs)
        await self._client.__aenter__()
        return self._client
```

**单例模式**: 确保每个 MCPClientManager 只创建一个客户端连接。

### 5.10.3 MCPToolSelector 类

```python
class MCPToolSelector:
    def __init__(self, cfg, researcher):
        self.cfg = cfg
        self.researcher = researcher
    
    async def select_tools(self, query, all_tools, max_tools=3):
        # 构建工具选择提示词
        prompt = self.researcher.prompt_family.generate_mcp_tool_selection_prompt(
            query, all_tools, max_tools
        )
        
        # 调用 LLM 选择工具
        response = await create_chat_completion(...)
        
        # 解析响应
        selected = json.loads(response)
        return selected["selected_tools"]
```

### 5.10.4 MCPResearchSkill 类

```python
class MCPResearchSkill:
    def __init__(self, cfg, researcher):
        self.cfg = cfg
        self.researcher = researcher
    
    async def execute_research(self, query, selected_tools):
        # 构建研究提示词
        prompt = self.researcher.prompt_family.generate_mcp_research_prompt(
            query, selected_tools
        )
        
        # 绑定工具到 LLM
        llm_with_tools = self.researcher.llm.bind_tools(selected_tools)
        
        # 执行研究
        response = await llm_with_tools.ainvoke([HumanMessage(content=prompt)])
        return response
```

---

## 5.11 Context 模块走读 (gpt_researcher/context/)

### 5.11.1 compression.py

#### 5.11.1.1 ContextCompressor 类

```python
class ContextCompressor:
    """使用嵌入相似度过滤文档块"""
    
    def __init__(self, documents, embeddings, max_results=5, similarity_threshold=None, ...):
        self.documents = documents
        self.embeddings = embeddings
        self.max_results = max_results
        self.similarity_threshold = similarity_threshold
    
    async def async_get_context(self, query, max_results=5, min_score=0.7):
        # 分割文档
        # 计算嵌入相似度
        # 过滤并返回最相关的块
        ...
```

**用途**: 从大量文档中提取与查询最相关的部分。

#### 5.11.1.2 VectorstoreCompressor 类

```python
class VectorstoreCompressor:
    """从向量存储中检索上下文"""
    
    async def async_get_context(self, query, max_results=5):
        results = await self.vector_store.asimilarity_search(query=query, k=max_results, filter=self.filter)
        return self.prompt_family.pretty_print_docs(results)
```

### 5.11.2 retriever.py

#### 5.11.2.1 SearchAPIRetriever 类

```python
class SearchAPIRetriever(BaseRetriever):
    pages: List[Dict] = []
    
    def _get_relevant_documents(self, query, *, run_manager):
        docs = [
            Document(
                page_content=(page.get("raw_content") or "")[:_MAX_CONTENT_CHARS],
                metadata={"title": page.get("title", ""), "source": page.get("url", "")},
            )
            for page in self.pages
        ]
        return docs
```

**常量**: `_MAX_CONTENT_CHARS = 50000` — 防止嵌入 API 超限。

#### 5.11.2.2 SectionRetriever 类

```python
class SectionRetriever(BaseRetriever):
    sections: List[Dict] = []
    
    def _get_relevant_documents(self, query, *, run_manager):
        docs = [
            Document(
                page_content=page.get("written_content", ""),
                metadata={"section_title": page.get("section_title", "")},
            )
            for page in self.sections
        ]
        return docs
```

**用途**: 检索已写内容，避免子话题报告重复。

---

## 5.12 Memory 和 VectorStore 模块

### 5.12.1 Memory 类 (embeddings.py)

```python
class Memory:
    def __init__(self, embedding_provider: str, model: str, **embedding_kwargs):
        match embedding_provider:
            case "openai":
                from langchain_openai import OpenAIEmbeddings
                _embeddings = OpenAIEmbeddings(model=model, **embedding_kwargs)
            case "ollama":
                from langchain_ollama import OllamaEmbeddings
                _embeddings = OllamaEmbeddings(base_url=..., **embedding_kwargs)
            # ... 15 更多提供商
        self._embeddings = _embeddings
    
    def get_embeddings(self):
        return self._embeddings
```

**设计**: 统一的嵌入提供商接口，懒加载依赖。

### 5.12.2 VectorStoreWrapper 类

```python
class VectorStoreWrapper:
    def __init__(self, vector_store: VectorStore):
        self.vector_store = vector_store
    
    def load(self, documents):
        langchain_documents = self._create_langchain_documents(documents)
        splitted_documents = self._split_documents(langchain_documents)
        self.vector_store.add_documents(splitted_documents)
    
    def _split_documents(self, documents, chunk_size=1000, chunk_overlap=200):
        text_splitter = RecursiveCharacterTextSplitter(
            chunk_size=chunk_size, chunk_overlap=chunk_overlap
        )
        return text_splitter.split_documents(documents)
    
    async def asimilarity_search(self, query, k, filter):
        return await self.vector_store.asimilarity_search(query=query, k=k, filter=filter)
```

**功能**: 文档分块、加载、相似度搜索。

---

## 5.13 Utils 模块走读 (gpt_researcher/utils/)

### 5.13.1 costs.py

#### 5.13.1.1 定价表

```python
OPENAI_MODEL_PRICING = (
    (("gpt-5.5-pro",), 30.0, 180.0),
    (("gpt-5.5",), 5.0, 30.0),
    (("gpt-5.4-pro",), 30.0, 180.0),
    (("gpt-5.4-mini",), 0.75, 4.5),
    (("gpt-5.4",), 2.5, 15.0),
    (("gpt-4.1",), 2.0, 8.0),
    (("gpt-4o",), 2.5, 10.0),
    (("o3",), 2.0, 8.0),
    ...
)

ANTHROPIC_MODEL_PRICING = (
    (("claude-opus-4-7",), 5.0, 25.0),
    (("claude-sonnet-4-6",), 3.0, 15.0),
    (("claude-haiku-4-5",), 1.0, 5.0),
    ...
)
```

#### 5.13.1.2 `calculate_llm_cost` 函数

```python
def calculate_llm_cost(
    llm_provider: str | None,
    model: str | None,
    input_content: str,
    output_content: str,
    response_metadata: Mapping[str, Any] | None = None,
    usage_metadata: Mapping[str, Any] | Any | None = None,
    request_options: Mapping[str, Any] | None = None,
) -> float:
    if llm_provider == "anthropic":
        anthropic_cost = calculate_anthropic_cost(...)
        if anthropic_cost is not None:
            return anthropic_cost
    
    # 优先使用 API 报告的 token 用量
    usage_tokens = _extract_usage_tokens(usage_metadata)
    if usage_tokens is not None:
        input_tokens, output_tokens = usage_tokens
        pricing = _get_openai_pricing(model)
        if pricing is not None:
            return (input_tokens * input_price + output_tokens * output_price) / 1_000_000
    
    # 回退到 tiktoken 估算
    return estimate_llm_cost(input_content, output_content)
```

**优先级**: Anthropic 特殊处理 → API 用量 → tiktoken 估算

### 5.13.2 llm.py

#### 5.13.2.1 `create_chat_completion` 函数

```python
async def create_chat_completion(
    messages: list[dict[str, str]],
    model: str | None = None,
    temperature: float | None = 0.4,
    max_tokens: int | None = 4000,
    llm_provider: str | None = None,
    stream: bool = False,
    websocket: Any | None = None,
    llm_kwargs: dict[str, Any] | None = None,
    cost_callback: callable = None,
    reasoning_effort: str | None = ReasoningEfforts.Medium.value,
    **kwargs
) -> str:
```

**功能**: 统一的 LLM 调用入口

**关键逻辑**:
1. 验证 `max_tokens` 不超过 200,000
2. 根据模型类型设置 `reasoning_effort` 和 `temperature`
3. 创建 `GenericLLMProvider` 实例
4. 调用 `get_chat_response()`
5. 失败时最多重试 10 次

**重试策略**:
```python
max_attempts = 1 if (stream and websocket is not None) else 10
for attempt in range(1, max_attempts + 1):
    try:
        response = await provider.get_chat_response(messages, stream, websocket, **kwargs)
    except Exception as exc:
        last_exception = exc
        logging.warning(f"LLM request failed (attempt {attempt}/{max_attempts}): {exc}")
```

### 5.13.3 workers.py (WorkerPool)

```python
class WorkerPool:
    def __init__(self, max_workers: int, rate_limit_delay: float = 0.0):
        self.max_workers = max_workers
        self.executor = ThreadPoolExecutor(max_workers=max_workers)
        self.semaphore = asyncio.Semaphore(max_workers)
        global_limiter = get_global_rate_limiter()
        global_limiter.configure(rate_limit_delay)
    
    @asynccontextmanager
    async def throttle(self):
        async with self.semaphore:
            global_limiter = get_global_rate_limiter()
            await global_limiter.wait_if_needed()
            yield
```

**双重控制**:
- `Semaphore`: 限制并发数
- `GlobalRateLimiter`: 全局限速

### 5.13.4 rate_limiter.py (GlobalRateLimiter)

```python
class GlobalRateLimiter:
    _instance: ClassVar['GlobalRateLimiter'] = None
    
    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance
    
    async def wait_if_needed(self):
        if self.rate_limit_delay <= 0:
            return
        async with self.get_lock():
            current_time = time.time()
            time_since_last = current_time - self.last_request_time
            if time_since_last < self.rate_limit_delay:
                await asyncio.sleep(self.rate_limit_delay - time_since_last)
            self.last_request_time = time.time()
```

**单例模式**: 确保全局只有一个限速器实例。

---

## 5.14 代码走读总结

### 5.14.1 架构模式总结

| 模式 | 应用位置 | 效果 |
|------|---------|------|
| 组合模式 | Agent + Skills | 灵活组装 |
| 工厂模式 | get_retriever, from_provider | 插件化 |
| 策略模式 | Retriever/Scraper 实现 | 可替换 |
| 模板方法 | Skills 继承 | 流程复用 |
| 观察者 | WebSocket 流式 | 实时通知 |
| 单例 | GlobalRateLimiter | 全局唯一 |
| 外观 | GPTResearcher | 简化接口 |

### 5.14.2 关键文件行数统计

| 文件 | 行数 | 职责 |
|------|------|------|
| agent.py | 794 | 主编排 |
| prompts.py | 903 | 提示词 |
| skills/researcher.py | 1082 | 研究执行 |
| skills/image_generator.py | 771 | 图像生成 |
| skills/deep_research.py | 646 | 深度研究 |
| llm_provider/generic/base.py | 420 | LLM 适配器 |
| retrievers/mcp/retriever.py | 323 | MCP 检索 |
| utils/tools.py | 349 | 工具调用 |

### 5.14.3 潜在改进点

1. **Agent 类过大**: 建议拆分为 AgentOrchestrator + AgentConfig + AgentSkills
2. **全局状态**: GlobalRateLimiter 的单例模式在测试中难以重置
3. **错误处理**: 部分函数返回空值而非抛出异常，可能导致下游难以调试
4. **类型安全**: 部分函数使用 `Any` 类型，可进一步精确化

---

> **下一章**: → `06-data-model.md` — 数据模型与数据库设计

---

☕️ 制作不易，请我喝咖啡☕️关注我➕