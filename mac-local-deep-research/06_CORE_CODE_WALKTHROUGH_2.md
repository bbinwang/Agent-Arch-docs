# 第 5 章（下）：核心代码讲解

> 本章延续第 5 章（上）的核心模块分析，聚焦六大子系统：LLM 提供商抽象层、嵌入与向量存储、内容提取与文档加载、引用处理系统、报告生成器、文本优化与错误处理。每个模块均包含文件路径、类/函数签名、核心逻辑逐步分析、设计模式识别、潜在问题与 Mermaid 图表。

---

## 目录

- [5.8 LLM 提供商抽象层](#58-llm-提供商抽象层)
- [5.9 嵌入与向量存储](#59-嵌入与向量存储)
- [5.10 内容提取与文档加载](#510-内容提取与文档加载)
- [5.11 引用处理系统](#511-引用处理系统)
- [5.12 报告生成器](#512-报告生成器)
- [5.13 文本优化与处理](#513-文本优化与处理)
- [5.14 错误处理与重试机制](#514-错误处理与重试机制)

---

## 5.8 LLM 提供商抽象层

**目录**：`src/local_deep_research/llm/providers/`

Local Deep Research 支持 14 个 LLM 提供商，通过统一的抽象层屏蔽底层差异。整个子系统采用"抽象基类 + OpenAI 兼容基类 + 具体实现 + 自动发现注册表"四层架构。

### 5.8.1 文件结构

```
llm/providers/
├── __init__.py                  # 包初始化
├── base.py                      # BaseLLMProvider 抽象基类 (185 行)
├── openai_base.py               # OpenAICompatibleProvider (417 行)
├── auto_discovery.py            # ProviderDiscovery 自动发现 (307 行)
├── _helpers.py                  # 上下文窗口与 max_tokens 计算
└── implementations/
    ├── openai.py                # OpenAIProvider
    ├── anthropic.py             # AnthropicProvider (原生)
    ├── google.py                # GoogleProvider
    ├── ollama.py                # OllamaProvider (本地)
    ├── lmstudio.py              # LMStudioProvider (本地)
    ├── deepseek.py              # DeepSeekProvider
    ├── openrouter.py            # OpenRouterProvider
    ├── xai.py                   # xAIProvider
    ├── ionos.py                 # IONOSProvider
    ├── requesty.py              # RequestyProvider
    ├── orcarouter.py            # OrcaRouterProvider
    ├── llamacpp.py              # LlamaCppProvider (本地)
    ├── custom_openai_endpoint.py  # 自定义 OpenAI 端点
    └── custom_anthropic_endpoint.py # 自定义 Anthropic 端点
```

### 5.8.2 BaseLLMProvider 抽象基类

**文件**：`llm/providers/base.py`（185 行）

`BaseLLMProvider` 定义了所有 LLM 提供商必须满足的最小接口契约。

#### 核心类属性

```python
class BaseLLMProvider:
    api_key_setting: str | None = None       # 设置键名，如 "llm.openai.api_key"
    provider_name: str = "unknown"           # 显示名称，子类必须覆盖
    api_key_optional: bool = False           # True 时缺失 key 返回 None 而非异常
    egress_exposure = Exposure.EXPOSING      # ADR-0007 出口暴露声明
```

#### 核心方法签名

```python
@classmethod
def create_llm(cls, model_name=None, temperature=0.7, **kwargs) -> BaseChatModel:
    """创建并返回 LangChain 聊天模型实例。
    契约：返回 BARE（未经包装的）BaseChatModel。
    包装（速率限制、token 计数、think 标签剥离）由 llm_config.get_llm() 在此返回后应用。
    子类不得在 create_llm() 内部包装。"""

@classmethod
def is_available(cls, settings_snapshot=None) -> bool:
    """默认返回 False（fail-closed）。子类必须覆盖。"""

@classmethod
def resolve_api_key(cls, settings_snapshot=None) -> str | None:
    """读取并规范化 API key。可选 provider 缺失返回 None；必须 provider 缺失抛 ValueError。"""

@classmethod
def resolve_api_key_or_placeholder(cls, settings_snapshot=None) -> str:
    """始终返回字符串。真实 key 缺失时返回 OPTIONAL_API_KEY_PLACEHOLDER ("not-required")。"""

@classmethod
def build_bearer_header(cls, settings_snapshot=None) -> dict[str, str]:
    """构建 Authorization: Bearer 头。无 key 时返回空 dict。"""

@classmethod
def has_api_key(cls, settings_snapshot=None) -> bool:
    """是否有真实 key 配置。吞异常返回 False，fail-closed。"""
```

#### 设计模式识别

- **模板方法模式**：`create_llm()` 定义骨架，子类填充具体实现。
- **类方法工厂模式**：所有方法为 `@classmethod`，无需实例化即可调用。
- **Fail-Closed 安全默认**：`is_available()` 默认返回 `False`，`has_api_key()` 吞异常返回 `False`。

#### 安全设计亮点

1. **API Key 占位符统一**：`OPTIONAL_API_KEY_PLACEHOLDER = "not-required"` 常量确保本地 provider（LM Studio、llama.cpp）在无需认证时不会因三个不同占位符字符串产生漂移。
2. **出口暴露声明（ADR-0007）**：`egress_exposure` 属性文档化意图，运行时由出口解析器根据实际端点细化分类。
3. **settings_snapshot 线程安全**：所有方法接受 `settings_snapshot` 参数，避免直接读取全局状态。

### 5.8.3 OpenAICompatibleProvider 实现

**文件**：`llm/providers/openai_base.py`（417 行）

这是最大的中间层基类，为任何提供 OpenAI 兼容 API 的服务（Google、OpenRouter、DeepSeek 等）提供通用实现。

#### 核心类属性

```python
class OpenAICompatibleProvider(BaseLLMProvider):
    provider_key: str              # 自动发现键名，如 "OPENAI"
    url_setting = None             # 可配置 URL 的设置键
    default_base_url = "https://api.openai.com/v1"
    default_model = ""             # 必须显式配置，无静默回退
```

#### create_llm() 核心逻辑

```python
@classmethod
def create_llm(cls, model_name=None, temperature=0.7, **kwargs):
    # 1. 解析 API key（必须 provider 缺失时抛 ValueError）
    api_key = cls.resolve_api_key_or_placeholder(settings_snapshot)

    # 2. 要求显式模型名——无静默回退到硬编码默认值
    if not model_name or not model_name.strip():
        raise ValueError(f"{cls.provider_name} model not configured.")

    # 3. 解析并校验 base URL
    base_url = kwargs.get("base_url", cls.default_base_url)
    base_url = normalize_url(base_url)

    # 4. SSRF 防护（仅 operator 可配置 URL 时启用）
    if cls.url_setting:
        base_url = assert_base_url_safe(base_url, setting_key=cls.url_setting)

    # 5. 计算上下文窗口感知的 max_tokens 上限（80% 上下文窗口）
    context_window_size = get_context_window_for_provider(cls.provider_key.lower())
    max_tokens = compute_max_tokens(context_window_size=context_window_size)

    # 6. 构建 ChatOpenAI 参数
    llm_params = {
        "model": model_name,
        "api_key": api_key,
        "base_url": base_url,
        "temperature": temperature,
        "max_tokens": max_tokens,
    }

    # 7. 添加可选参数（streaming、max_retries、request_timeout）
    _get_optional_setting(llm_params, "streaming", "llm.streaming", ...)
    _get_optional_setting(llm_params, "max_retries", "llm.max_retries", ...)
    _get_optional_setting(llm_params, "request_timeout", "llm.request_timeout", ...)

    return ChatOpenAI(**llm_params)
```

#### SSRF 防护机制

```python
if cls.url_setting:
    base_url = assert_base_url_safe(base_url, setting_key=cls.url_setting)
```

**关键安全设计**：
- 仅当 `url_setting` 不为 None 时才执行 SSRF 校验（OpenAI/Anthropic 等硬编码 URL 的 provider 无需校验）
- `ALWAYS_BLOCKED_METADATA_IPS` 始终生效，即使在校验标志较宽松的云模式下
- 校验覆盖 `create_llm()`、`list_models_for_api()` 两条路径

#### list_models_for_api() 模型列表

```python
@classmethod
def list_models_for_api(cls, api_key=None, base_url=None):
    # 纵深防御：拒绝非字符串 api_key（防止 dict 被 repr 泄露）
    if api_key is not None and not isinstance(api_key, str):
        return []

    # SSRF 校验 base_url
    if base_url and cls.url_setting:
        base_url = assert_base_url_safe(base_url, setting_key=cls.url_setting)

    client = OpenAI(api_key=api_key, base_url=base_url)
    models_response = client.models.list()
    return [{"value": m.id, "label": m.id} for m in models_response.data]
```

### 5.8.4 ProviderDiscovery 自动发现机制

**文件**：`llm/providers/auto_discovery.py`（307 行）

#### 单例模式实现

```python
class ProviderDiscovery:
    _instance = None
    _providers: Dict[str, ProviderInfo] = {}
    _discovered: bool = False

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance
```

#### discover_providers() 扫描算法

```python
def discover_providers(self, force_refresh=False):
    if self._discovered and not force_refresh:
        return self._providers

    implementations_dir = Path(__file__).parent / "implementations"

    for file_path in implementations_dir.glob("*.py"):
        if file_path.name.startswith("_"):
            continue

        module = importlib.import_module(
            f".implementations.{module_name}",
            package="local_deep_research.llm.providers"
        )

        for name, obj in inspect.getmembers(module, inspect.isclass):
            # 注册条件：
            # 1. 类名以 "Provider" 结尾
            # 2. provider_name 非空非 "unknown"
            # 3. 是 BaseLLMProvider 子类
            # 4. 不是 OpenAICompatibleProvider/BaseLLMProvider 本身
            # 5. 类必须定义在该模块内（非仅导入）
            if (name.endswith("Provider")
                and getattr(obj, "provider_name", "unknown") not in (None, "", "unknown")
                and issubclass(obj, BaseLLMProvider)
                and obj is not OpenAICompatibleProvider
                and obj is not BaseLLMProvider
                and obj.__module__ == module.__name__):
                provider_info = ProviderInfo(obj)
                self._providers[provider_info.provider_key] = provider_info
                register_llm(normalize_provider(provider_info.provider_key), obj.create_llm)

    self._discovered = True
    return self._providers
```

**关键过滤逻辑**：`obj.__module__ == module.__name__` 防止导入的基类被重复注册（如 `custom_anthropic_endpoint.py` 导入 `AnthropicProvider` 作为基类时，不将其注册为独立 provider）。

#### ProviderInfo 元数据

```python
class ProviderInfo:
    provider_class      # 提供商类
    provider_key        # 大写键名（如 "OPENAI"）
    provider_name       # 显示名（如 "OpenAI"）
    company_name        # 公司名
    is_cloud            # 云/本地标识
    display_name        # 带图标的显示名（如 "OpenAI ☁️ Cloud"）
```

### 5.8.5 LLMRegistry 线程安全注册表

**文件**：`llm/llm_registry.py`（168 行）

```python
class LLMRegistry:
    def __init__(self):
        self._llms: Dict[str, BaseChatModel | Callable[..., BaseChatModel]] = {}
        self._lock = threading.Lock()

    def register(self, name, llm):
        with self._lock:
            normalized_name = name.lower()
            self._llms[normalized_name] = llm

    def get(self, name):
        with self._lock:
            return self._llms.get(name.lower())

    def clear(self):
        with self._lock:
            self._llms.clear()
```

**设计要点**：
- `threading.Lock` 保证多线程安全（Flask 多线程环境）
- 名称统一小写规范化，支持大小写不敏感查找
- `clear()` 被 64+ 测试依赖（测试隔离）

### 5.8.6 14 个提供商分类

| 分类 | 提供商 | 实现方式 | API Key | 本地/云 |
|------|--------|----------|---------|---------|
| OpenAI 兼容（云） | OpenAI, Google, DeepSeek, OpenRouter, xAI, IONOS, Requesty, OrcaRouter | `OpenAICompatibleProvider` 子类 | 必须 | 云 |
| OpenAI 兼容（本地） | LM Studio, llama.cpp, Custom OpenAI Endpoint | `OpenAICompatibleProvider` 子类 | 可选 | 本地 |
| 原生 SDK | Anthropic, Custom Anthropic Endpoint | 直接调用 Anthropic SDK | 必须 | 云 |

### 5.8.7 Mermaid 类图

```mermaid
classDiagram
    class BaseLLMProvider {
        <<abstract>>
        +api_key_setting: str|None
        +provider_name: str
        +api_key_optional: bool
        +egress_exposure: Exposure
        +create_llm(model_name, temperature, **kwargs) BaseChatModel*
        +is_available(settings_snapshot) bool*
        +resolve_api_key(settings_snapshot) str|None
        +resolve_api_key_or_placeholder(settings_snapshot) str
        +build_bearer_header(settings_snapshot) dict
        +has_api_key(settings_snapshot) bool
        +requires_auth_for_models() bool
    }

    class OpenAICompatibleProvider {
        +provider_key: str
        +url_setting: str|None
        +default_base_url: str
        +create_llm(model_name, temperature, **kwargs) ChatOpenAI
        +list_models_for_api(api_key, base_url) List
        +list_models(settings_snapshot) List
        -_create_llm_instance(**kwargs) ChatOpenAI
        -_get_base_url_for_models(settings_snapshot) str
    }

    class OpenAIProvider {
        +provider_name = "OpenAI"
        +api_key_setting = "llm.openai.api_key"
        +default_base_url = "https://api.openai.com/v1"
    }

    class AnthropicProvider {
        +provider_name = "Anthropic"
        +create_llm(**kwargs) ChatAnthropic
    }

    class OllamaProvider {
        +provider_name = "Ollama"
        +api_key_optional = True
    }

    class LMStudioProvider {
        +provider_name = "LM Studio"
        +api_key_optional = True
        +url_setting = "llm.lmstudio.url"
    }

    class ProviderDiscovery {
        -_instance: ProviderDiscovery
        -_providers: Dict~str,ProviderInfo~
        -_discovered: bool
        +discover_providers(force_refresh) Dict
        +get_provider_info(key) ProviderInfo
        +get_provider_options() List
        +get_available_provider_options(snapshot) List
        +get_provider_class(key) class
    }

    class ProviderInfo {
        +provider_class: class
        +provider_key: str
        +provider_name: str
        +is_cloud: bool
        +display_name: str
        +to_dict() dict
    }

    class LLMRegistry {
        -_llms: Dict~str,BaseChatModel|Callable~
        -_lock: threading.Lock
        +register(name, llm) void
        +unregister(name) void
        +get(name) BaseChatModel|Callable|None
        +is_registered(name) bool
        +list_registered() List~str~
        +clear() void
    }

    BaseLLMProvider <|-- OpenAICompatibleProvider
    BaseLLMProvider <|-- AnthropicProvider
    OpenAICompatibleProvider <|-- OpenAIProvider
    OpenAICompatibleProvider <|-- LMStudioProvider
    OpenAICompatibleProvider <|-- OllamaProvider
    ProviderDiscovery --> ProviderInfo : creates
    ProviderDiscovery --> BaseLLMProvider : discovers
    ProviderDiscovery --> LLMRegistry : register_llm()
    LLMRegistry o-- BaseChatModel : stores
```

**类图说明**：该图展示了 LLM 提供商抽象层的完整类层次结构。`BaseLLMProvider` 作为根抽象基类定义了所有 provider 必须实现的契约。`OpenAICompatibleProvider` 是其最大子类，封装了 OpenAI API 兼容服务的通用逻辑（SSRF 防护、ChatOpenAI 参数构建、模型列表获取）。具体 provider（OpenAI、Anthropic、Ollama、LM Studio 等）通过继承这两个基类实现差异化。`ProviderDiscovery` 作为单例负责在 `implementations/` 目录下扫描所有 provider 类，提取元数据并自动注册到 `LLMRegistry` 全局注册表。`ProviderInfo` 封装每个 provider 的显示元数据（键名、公司名、云/本地标识、显示名称），供 UI 下拉框使用。

### 5.8.8 潜在问题与改进点

1. **ProviderDiscovery 单例测试隔离**：`_discovered` 标志在测试间不重置，可能导致测试间状态泄漏。建议在测试 fixture 中调用 `ProviderDiscovery._instance = None`。
2. **API Key 占位符语义**：`"not-required"` 字符串作为 API key 可能在某些严格校验的 SDK 版本中被拒绝。
3. **LLMRegistry 类型擦除**：存储的 `BaseChatModel | Callable` 丢失了具体 provider 元数据，调试时难以区分来源。

---

## 5.9 嵌入与向量存储

**目录**：`src/local_deep_research/embeddings/`、`src/local_deep_research/vector_stores/`

向量存储子系统采用"门面模式 + 仓库模式"双层架构：`VectorIndex` 作为统一入口，底层 `FaissVectorStore` 驱动原生 FAISS 库，`DocumentChunk` 作为加密 DB 中的权威文本存储。

### 5.9.1 文件结构

```
embeddings/
├── __init__.py
├── embeddings_config.py         # 嵌入配置
├── providers/
│   ├── base.py                  # BaseEmbeddingProvider (167 行)
│   └── implementations/
│       ├── openai.py            # OpenAIEmbeddingProvider
│       ├── ollama.py            # OllamaEmbeddingProvider
│       └── sentence_transformers.py  # SentenceTransformersProvider
└── splitters/
    └── text_splitter_registry.py  # 分块策略注册表

vector_stores/
├── __init__.py
├── base.py                      # BaseVectorStore 抽象基类
├── facade.py                    # VectorIndex 门面 (525 行)
├── config.py                    # 向量存储配置
├── legacy_rekey.py              # 旧 UUID key → int64 key 迁移
├── legacy_cleanup.py            # 遗留清理
└── implementations/
    └── faiss_store.py           # FaissVectorStore (1043 行)
```

### 5.9.2 BaseEmbeddingProvider 抽象

**文件**：`embeddings/providers/base.py`（167 行）

```python
class BaseEmbeddingProvider(ABC):
    provider_name = "base"
    provider_key = "BASE"
    requires_api_key = False
    supports_local = False
    default_model = None
    egress_exposure = Exposure.EXPOSING

    @classmethod
    @abstractmethod
    def create_embeddings(cls, model, settings_snapshot, **kwargs) -> Embeddings:
        """创建 LangChain Embeddings 实例。"""

    @classmethod
    @abstractmethod
    def is_available(cls, settings_snapshot) -> bool:
        """检查 provider 是否可用并正确配置。"""

    @classmethod
    def get_available_models(cls, settings_snapshot) -> List[Dict[str, Any]]:
        """获取可用模型列表。默认返回空列表。"""

    @classmethod
    def is_embedding_model(cls, model, settings_snapshot) -> Optional[bool]:
        """检查模型是否支持嵌入。返回 True/False/None（无法判断）。"""

    @classmethod
    def validate_config(cls, settings_snapshot) -> tuple[bool, Optional[str]]:
        """验证配置。返回 (is_valid, error_message)。"""
```

### 5.9.3 三个嵌入提供商实现对比

| 特性 | SentenceTransformers | Ollama | OpenAI |
|------|---------------------|--------|--------|
| 本地运行 | ✅ 完全本地 | ✅ 本地服务 | ❌ 云服务 |
| 依赖 | `sentence-transformers` + PyTorch | Ollama 服务 | OpenAI API Key |
| 默认模型 | `all-MiniLM-L6-v2` | `nomic-embed-text` | `text-embedding-3-small` |
| 维度 | 384 | 768 | 1536 |
| 出口暴露 | CONTAINED | CONTAINED | EXPOSING |
| 自动发现 | ✅ | ✅ | ✅ |

### 5.9.4 FaissVectorStore 实现细节

**文件**：`vector_stores/implementations/faiss_store.py`（1043 行）

这是整个项目最复杂的单个文件之一，直接驱动 Facebook FAISS 库。

#### 安全不变量

```
SECURITY INVARIANT：向量存储仅持有向量 + int64 id。
永不存储文档文本、元数据。所有文本/元数据由加密 DB 中的
DocumentChunk 权威持有，搜索时通过 id 从 DB 重新水化。
```

#### IndexIDMap2 键设计

```python
class FaissVectorStore(BaseVectorStore):
    def add(self, ids: List[int], vectors: np.ndarray):
        id_arr = np.asarray(list(ids), dtype="int64")
        vecs = self._prepare(vectors)
        self._index.add_with_ids(vecs, id_arr)  # int64 键 = DocumentChunk.id
```

**关键设计**：使用 `IndexIDMap2` 而非 `IndexFlatL2`，将 id 管理交给 FAISS C++ 层，避免 Python 侧位置映射和删除时的编号重排。

#### apply() 原子操作

```python
def apply(self, *, add_ids, add_vectors, remove_ids=(), dedup=True) -> dict:
    """持久化批量写入。流程：
    1. 获取写锁
    2. _reload_under_lock() — 合并其他写入者的保存
    3. 删除（旧向量 + 碰撞的 live id）
    4. 去重（跨批次 + 已存在的 live id）
    5. 添加新向量
    6. persist() — 原子写入（temp + os.replace）
    7. _record_integrity() — 记录完整性校验和
    """
```

**HNSW 特殊处理**：HNSW 索引没有 `remove_ids` 实现，删除触发全量重建。`_apply_hnsw()` 采用三阶段无锁算法：
1. Phase 1：在锁内快照 live ids + 重建向量
2. Phase 2：在锁外构建新图（O(N) 最昂贵操作）
3. Phase 3：仅在文件未变时交换（版本检查），否则重试

#### persist() 原子写入

```python
def persist(self, path: Path):
    # 1. 创建私有目录（0o700）
    path.parent.chmod(0o700)

    # 2. 清理陈旧 tmp 文件（>1 小时）

    # 3. 写入 temp 文件（O_EXCL | 随机名）
    fd, tmp_name = tempfile.mkstemp(dir=prefix="name.", suffix=".tmp")
    write_index(self._index, str(tmp))

    # 4. fsync temp 字节
    os.fsync(f.fileno())

    # 5. 原子替换
    os.replace(tmp, path)

    # 6. fsync 目录（保证 rename 持久化）
```

**崩溃安全**：temp + os.replace 关闭写撕裂窗口（issue #4197）。并发读取者要么看到旧字节，要么看到新字节，永远不会看到截断混合。

#### _prepare() 向量校验

```python
def _prepare(self, vectors):
    vecs = np.array(vectors, dtype="float32", copy=True, order="C")

    # 显式维度校验（FAISS 仅在 assert 中校验，-O 时被剥离）
    if vecs.shape[1] != self.dimension:
        raise ValueError(...)

    # 拒绝非有限输入（NaN/Inf 永久毒化索引）
    if not np.isfinite(vecs).all():
        raise ValueError("vectors contain non-finite values")

    if self.normalize:
        # L2 归一化前检查零向量（归一化后变为全零，不可搜索）
        norms = np.sqrt((vecs**2).sum(axis=1))
        if not np.isfinite(norms).all() or (norms == 0).any():
            raise ValueError(...)
        normalize_L2(vecs)
    else:
        # 检查 float32 溢出
        sq_norms = (vecs.astype("float64")**2).sum(axis=1)
        if (sq_norms > _MAX_SAFE_SQ_NORM).any():
            raise ValueError(...)

    return vecs
```

### 5.9.5 VectorIndex 门面

**文件**：`vector_stores/facade.py`（525 行）

`VectorIndex` 是向量子系统的唯一入口，桥接三个关注点：

1. **嵌入模型**（LangChain `Embeddings`）
2. **加密 DB**（`DocumentChunk` — 文本/元数据的权威持有者）
3. **可插拔向量存储**（`BaseVectorStore` 实现）

#### index() 流程

```python
def index(self, *, source_type, source_id, chunks, replace=True, session=None):
    # 1. 嵌入文本
    vectors = np.asarray(self.embeddings.embed_documents(texts), dtype="float32")

    # 2. 查询该 source 的前置 chunk ids（限定当前嵌入模型配置）
    prior_ids = [row.id for row in sess.query(DocumentChunk.id).filter_by(
        source_type=source_type, source_id=sid,
        collection_name=self.collection_name,
        embedding_model=self.embedding_model,        # 限定模型
        embedding_model_type=self.embedding_model_type,  # 限定类型
    )]

    # 3. 构建 DocumentChunk 行 + flush 获取 int PK
    rows = [self._build_row(c, source_type, sid) for c in chunks]
    sess.add_all(rows)
    sess.flush()  # 分配 int PK
    new_ids = [row.id for row in rows]

    # 4. store.apply() — 持久化向量（原子，在锁内）
    stats = self._store.apply(add_ids=new_ids, add_vectors=vectors, remove_ids=prior_ids)

    # 5. 删除前置行 + commit
    self._finalize_db(sess, owns, prior_ids, ...)
```

**关键安全设计**：前置 chunk 查询限定 `embedding_model` + `embedding_model_type`，防止重新索引模型 B 时破坏模型 A 的 chunk 文本（每个模型有独立的物理 .faiss 文件）。

#### search() 流程

```python
def search(self, query, top_k, session=None):
    # 1. 嵌入查询
    query_vec = np.asarray(self.embeddings.embed_query(query), dtype="float32")

    # 2. 在向量存储中搜索
    hits = self._store.search(query_vec, top_k)  # [(chunk_id, distance)]

    # 3. 限定当前集合，从 DB 重新水化
    rows = {row.id: row for row in sess.query(DocumentChunk).filter(
        DocumentChunk.id.in_(ids),
        DocumentChunk.collection_name == self.collection_name,  # 防止跨集合泄漏
    )}

    # 4. 构建 SearchResult（孤儿命中跳过）
    for chunk_id, distance in hits:
        row = rows.get(chunk_id)
        if row is None:
            continue  # 孤儿向量，跳过
        results.append(SearchResult(chunk_id, row.chunk_text, ...))
```

**跨集合隔离**：`collection_name` 过滤防止回滚插入遗留的孤儿向量匹配到不同集合的行，避免文本泄漏。

### 5.9.6 DocumentChunk 模型

**文件**：`database/models/library.py` 中的 `DocumentChunk`（约 80 行）

```python
class DocumentChunk(Base):
    __tablename__ = "document_chunks"

    id = Column(Integer, primary_key=True, autoincrement=True)
    chunk_hash = Column(String(64), nullable=False, index=True)  # SHA256 去重
    source_type = Column(String(20), nullable=False, index=True)  # document, folder_file
    source_id = Column(String(36), nullable=True, index=True)  # Document.id
    source_path = Column(Text, nullable=True)  # 本地文件路径
    collection_name = Column(String(100), nullable=False, index=True)
    chunk_text = Column(Text, nullable=False)  # 实际文本（加密存储）
    chunk_index = Column(Integer, nullable=False)
    start_char = Column(Integer, nullable=False)
    end_char = Column(Integer, nullable=False)
    word_count = Column(Integer, nullable=False)
    embedding_id = Column(String(36), nullable=False, unique=True, index=True)  # 旧 UUID 兼容
    embedding_model = Column(String(100), nullable=False)
    embedding_model_type = Column(Enum(EmbeddingProvider), nullable=False)
    embedding_dimension = Column(Integer, nullable=True)
    document_title = Column(Text, nullable=True)
    document_metadata = Column(JSON, nullable=True)

    __table_args__ = (
        Index("idx_chunk_source", "source_type", "source_id"),
        Index("idx_chunk_collection", "collection_name", "created_at"),
        Index("idx_chunk_embedding", "embedding_id"),
        {"sqlite_autoincrement": True},  # 关键：AUTOINCREMENT 防止 id 复用
    )
```

**AUTOINCREMENT 必要性**：SQLite ROWID 在行删除后会被回收复用。如果 `DocumentChunk.id` 被回收，新行可能继承旧 id，而旧向量仍残留在 FAISS 索引中，导致搜索返回错误文档的文本。AUTOINCREMENT 保证 id 单调递增，永不复用。

### 5.9.7 Mermaid 类图

```mermaid
classDiagram
    class VectorIndex {
        +username: str
        +db_password: str
        +embeddings: Embeddings
        +embedding_model: str
        +collection_name: str
        +metric: str
        +_store: BaseVectorStore
        +index(source_type, source_id, chunks, replace, session) IndexStats
        +search(query, top_k, session) List~SearchResult~
        +delete(source_type, source_id, session) DeleteStats
        +delete_ids(ids, session) DeleteStats
        +count() int
        -_db(session) Iterator
        -_build_row(chunk, source_type, source_id) DocumentChunk
    }

    class BaseVectorStore {
        <<abstract>>
        +provider_key: str
        +dimension: int
        +index_type: str
        +metric: str
        +normalize: bool
        +create(**kwargs) BaseVectorStore*
        +load(path, **kwargs) BaseVectorStore*
        +add(ids, vectors) void*
        +search(query_vector, k) List~Tuple[int,float]~*
        +delete(ids) int*
        +apply(add_ids, add_vectors, remove_ids) dict*
        +persist(path) void*
        +count() int
        +live_ids() List~int~
        +supports_delete: bool
    }

    class FaissVectorStore {
        -_index: IndexIDMap2
        -_lock: threading.Lock
        -_path: Path
        -_poisoned: bool
        -_integrity_record: Callable
        -_integrity_verify: Callable
        +create(**kwargs) FaissVectorStore
        +load(path, **kwargs) FaissVectorStore
        +apply(add_ids, add_vectors, remove_ids) dict
        -_apply_locked(add_ids, add_vectors, remove_ids, dedup) dict
        -_apply_hnsw(add_ids, add_vectors, remove_ids, dedup) dict
        -_reload_under_lock() void
        -_restore_from_disk() void
        -_rebuild_dropping(remove_ids) int
        -_dedup_new(ids, vectors) Tuple
        -_prepare(vectors) np.ndarray
        +persist(path) void
        +search(query_vector, k) List~Tuple[int,float]~
        +delete(ids) int
        +count() int
        +reconstruct(id) np.ndarray|None
    }

    class ChunkInput {
        +text: str
        +metadata: dict
    }

    class SearchResult {
        +chunk_id: int
        +text: str
        +distance: float
        +metric: str
        +metadata: dict
        +document_title: str|None
        +source_id: str|None
        +source_type: str|None
    }

    class DocumentChunk {
        +id: Integer (AUTOINCREMENT)
        +chunk_hash: String(64)
        +source_type: String(20)
        +source_id: String(36)
        +collection_name: String(100)
        +chunk_text: Text
        +chunk_index: Integer
        +embedding_id: String(36) UNIQUE
        +embedding_model: String(100)
        +embedding_model_type: EmbeddingProvider
        +document_title: Text
    }

    class IndexStats {
        +chunks: int
        +added: int
        +removed: int
    }

    class DeleteStats {
        +removed: int
        +rows_deleted: int
    }

    VectorIndex --> BaseVectorStore : uses
    BaseVectorStore <|-- FaissVectorStore
    VectorIndex ..> ChunkInput : accepts
    VectorIndex ..> SearchResult : returns
    VectorIndex ..> IndexStats : returns
    VectorIndex ..> DeleteStats : returns
    VectorIndex ..> DocumentChunk : reads/writes
    FaissVectorStore ..> DocumentChunk : ids only
```

**类图说明**：该图展示了向量存储子系统的完整类结构。`VectorIndex` 作为门面（Facade）是外部调用者的唯一入口，它组合了嵌入模型（`Embeddings`）、向量存储（`BaseVectorStore`）和加密 DB 会话。`BaseVectorStore` 定义了所有后端必须实现的接口契约，`FaissVectorStore` 是唯一的实现（Provider 名 "faiss"）。`DocumentChunk` 是 SQLAlchemy 模型，作为文本和元数据的权威存储（在 SQLCipher 加密 DB 中）。`ChunkInput`、`SearchResult`、`IndexStats`、`DeleteStats` 是数据传输对象。关键安全约束体现在：`FaissVectorStore` 仅通过 `DocumentChunk.id`（int64）引用文本，永不接收或存储文本内容。

### 5.9.8 向量索引流程图

```mermaid
flowchart TD
    A[调用者: index 请求] --> B[VectorIndex.index]
    B --> C[嵌入文本: embeddings.embed_documents]
    C --> D[查询该 source 的前置 chunk ids]
    D --> E{replace=true?}
    E -->|是| F[查询同 source + 同 model 的前置 ids]
    E -->|否| G[无前置 ids]
    F --> H[构建 DocumentChunk 行]
    G --> H
    H --> I[sess.add_all + flush 获取 int PK]
    I --> J[_store.apply: add + remove]
    J --> K{index_type?}
    K -->|flat/ivf| L[_apply_locked: 全锁路径]
    K -->|hnsw| M[_apply_hnsw: 三阶段无锁]
    L --> N[persist: temp + os.replace]
    M --> N
    N --> O[_record_integrity]
    O --> P[_finalize_db: 删除旧行 + commit]
    P --> Q[返回 IndexStats]

    R[调用者: search 请求] --> S[VectorIndex.search]
    S --> T[嵌入查询: embeddings.embed_query]
    T --> U[_store.search: FAISS 搜索]
    U --> V[过滤: collection_name + id IN]
    V --> W[从 DB 重新水化 chunk_text]
    W --> X{row 存在?}
    X -->|是| Y[构建 SearchResult]
    X -->|否| Z[跳过孤儿命中]
    Z --> AA[返回结果列表]
    Y --> AA
```

**流程图说明**：该图展示了向量索引的 `index` 和 `search` 两条核心路径。`index` 路径（左侧）从嵌入文本开始，先查询并锁定该 source 的前置 chunk ids（限定当前嵌入模型配置），然后 flush 获取新行的 int PK，接着调用 `store.apply()` 原子化持久化向量（flat/IVF 走全锁路径，HNSW 走三阶段无锁路径），最后删除旧 DB 行并提交。`search` 路径（右侧）从嵌入查询开始，在 FAISS 中搜索 top-k 命中，然后通过 `DocumentChunk.id` 从加密 DB 重新水化文本（限定 `collection_name` 防止跨集合泄漏），孤儿命中（DB 行已删除但向量仍存在）被安全跳过。

### 5.9.9 潜在问题与改进点

1. **多进程部署竞态**：`_lock` 是 `threading.Lock`，仅在单进程内有效。`gunicorn -w N` 多 worker 部署时，两个 worker 持有独立锁，存在丢失更新风险。需文件级锁。
2. **HNSW 全量重建成本**：HNSW 删除触发 O(N) 全量重建，大集合（>100k chunks）可能耗时数秒。
3. **嵌入模型切换成本**：更换嵌入模型需要全量重新索引，无增量迁移路径。
4. **DocumentChunk.embedding_id 冗余**：为旧 UUID → int64 一次性迁移保留的列，增加了每行存储开销。

---

## 5.10 内容提取与文档加载

**目录**：`src/local_deep_research/content_fetcher/`、`src/local_deep_research/document_loaders/`、`src/local_deep_research/research_library/downloaders/`

内容提取子系统负责从各种来源（学术网站、通用网页、PDF 链接）获取并提取文本内容。

### 5.10.1 文件结构

```
content_fetcher/
├── __init__.py
├── url_classifier.py            # URLClassifier 分类器 (243 行)
└── fetcher.py                   # ContentFetcher 统一路由器 (463 行)

research_library/downloaders/
├── base.py                      # BaseDownloader 基类
├── arxiv.py                     # ArxivDownloader
├── pubmed.py                    # PubMedDownloader
├── semantic_scholar.py          # SemanticScholarDownloader
├── biorxiv.py                   # BioRxivDownloader
├── direct_pdf.py                # DirectPDFDownloader
└── playwright_html.py           # AutoHTMLDownloader (Playwright 渲染)

document_loaders/
├── pdf_loader.py                # PDF 加载器
├── docx_loader.py               # DOCX 加载器
├── pptx_loader.py               # PPTX 加载器
└── xlsx_loader.py               # XLSX 加载器
```

### 5.10.2 URLClassifier 分类算法

**文件**：`content_fetcher/url_classifier.py`（243 行）

#### URLType 枚举

```python
class URLType(Enum):
    ARXIV = "arxiv"
    PUBMED = "pubmed"
    PMC = "pmc"                   # PubMed Central
    SEMANTIC_SCHOLAR = "semantic_scholar"
    BIORXIV = "biorxiv"
    MEDRXIV = "medrxiv"
    DOI = "doi"
    PDF = "pdf"                   # 直接 PDF 链接
    HTML = "html"                 # 通用网页
    INVALID = "invalid"           # 危险/不支持的协议
```

#### 分类算法流程

```python
@classmethod
def classify(cls, url: str) -> URLType:
    url_lower = url.lower().strip()

    # 1. 安全检查：拒绝危险协议
    parsed = urlparse(url_lower)
    if parsed.scheme in DANGEROUS_SCHEMES:  # javascript, data, file, vbscript, about
        return URLType.INVALID
    if parsed.scheme and parsed.scheme not in ("http", "https", ""):
        return URLType.INVALID

    # 2. 检查直接 PDF 链接（排除学术域名）
    if cls._is_pdf_url(url_lower):
        return URLType.PDF

    # 3. 正则匹配学术来源（按优先级顺序）
    for pattern in cls.ARXIV_PATTERNS:       # arxiv.org/abs/, /pdf/, /html/, ar5iv.org
        if re.search(pattern, url_lower):
            return URLType.ARXIV
    for pattern in cls.PMC_PATTERNS:         # ncbi.nlm.nih.gov/pmc/articles/pmc
        if re.search(pattern, url_lower):
            return URLType.PMC
    for pattern in cls.PUBMED_PATTERNS:      # pubmed.ncbi.nlm.nih.gov/\d+
        if re.search(pattern, url_lower):
            return URLType.PUBMED
    for pattern in cls.SEMANTIC_SCHOLAR_PATTERNS:
        if re.search(pattern, url_lower):
            return URLType.SEMANTIC_SCHOLAR
    for pattern in cls.BIORXIV_PATTERNS:     # biorxiv.org/content/
        if re.search(pattern, url_lower):
            return URLType.BIORXIV
    for pattern in cls.MEDRXIV_PATTERNS:     # medrxiv.org/content/
        if re.search(pattern, url_lower):
            return URLType.MEDRXIV
    for pattern in cls.DOI_PATTERNS:         # doi.org/10., dx.doi.org/10.
        if re.search(pattern, url_lower):
            return URLType.DOI

    # 4. 默认 HTML
    return URLType.HTML
```

#### 安全约束

```python
DANGEROUS_SCHEMES = {"javascript", "data", "file", "vbscript", "about"}
```

**纵深防御**：URLClassifier 是第一道防线，ContentFetcher 在调用下载器前还会执行 `policy_aware_validate_url()` 和 `evaluate_url()` 进行 SSRF 和出口策略校验。

#### extract_id() 标识符提取

```python
@classmethod
def extract_id(cls, url, url_type=None) -> Optional[str]:
    if url_type == URLType.ARXIV:
        match = re.search(r"(\d{4}\.\d{4,5}(?:v\d+)?)", url)  # 新格式
        match = re.search(r"([a-z-]+/\d{7}(?:v\d+)?)", url)   # 旧格式
    elif url_type == URLType.PUBMED:
        match = re.search(r"/(\d+)/?", url)  # PMID
    elif url_type == URLType.PMC:
        match = re.search(r"PMC(\d+)", url, re.IGNORECASE)
    elif url_type == URLType.SEMANTIC_SCHOLAR:
        match = re.search(r"/paper/(?:[^/]+/)?([a-f0-9]{40})", url)  # 40-char hex
    elif url_type == URLType.DOI:
        match = re.search(r"(10\.\d{4,}/[^\s]+)", url)
```

### 5.10.3 ContentFetcher 统一获取路由器

**文件**：`content_fetcher/fetcher.py`（463 行）

#### 构造函数

```python
class ContentFetcher:
    def __init__(
        self,
        timeout: int = 30,
        language: str = "English",
        enable_js_rendering: bool = False,
        egress_context: Any = None,
    ):
        self.timeout = timeout
        self.language = language
        self.enable_js_rendering = enable_js_rendering
        self.egress_context = egress_context
        self._downloaders: Dict[URLType, Any] = {}  # 下载器缓存
```

#### _get_downloader() 懒加载

```python
def _get_downloader(self, url_type: URLType):
    if url_type in self._downloaders:
        return self._downloaders[url_type]

    downloader = None
    if url_type == URLType.ARXIV:
        from ..research_library.downloaders.arxiv import ArxivDownloader
        downloader = ArxivDownloader(timeout=self.timeout)
    elif url_type in (URLType.PUBMED, URLType.PMC):
        from ..research_library.downloaders.pubmed import PubMedDownloader
        downloader = PubMedDownloader(timeout=self.timeout)
    # ... 类似处理其他类型

    if downloader:
        self._apply_egress_policy_to_downloader(downloader)
        self._downloaders[url_type] = downloader
    return downloader
```

#### fetch() 核心流程

```python
def fetch(self, url, max_length=None, prefer_text=True):
    # 1. 分类 URL
    url_type = URLClassifier.classify(url)

    # 2. 拒绝无效 URL
    if url_type == URLType.INVALID:
        return {"status": "error", "error": "Invalid URL"}

    # 3. SSRF 校验（policy-aware）
    if not policy_aware_validate_url(url, self.egress_context):
        return {"status": "error", "error": "URL failed security validation"}

    # 4. 出口策略校验
    url_decision = evaluate_url(url, self.egress_context)
    if not url_decision.allowed:
        return {"status": "error", "error": "URL refused by egress policy"}

    # 5. 获取下载器
    downloader = self._get_downloader(url_type)

    # 6. 下载内容
    result = downloader.download_with_result(url, content_type)

    # 7. HTML 回退（专业下载器失败时）
    if not result.is_success and url_type not in _NO_HTML_FALLBACK:
        html_downloader = self._get_downloader(URLType.HTML)
        result = html_downloader.download_with_result(url, content_type)

    # 8. 解码内容（检测 PDF magic bytes）
    if result.content[:4] == b"%PDF":
        content = BaseDownloader.extract_text_from_pdf(result.content)
    else:
        content = result.content.decode("utf-8")

    # 9. 截断（默认 500KB）
    if max_length and len(content) > max_length:
        content = content[:max_length] + "\n\n[... content truncated ...]"

    return {"status": "success", "content": content, ...}
```

#### 出口策略应用到下载器

```python
def _apply_egress_policy_to_downloader(self, downloader):
    """当 scope=PRIVATE_ONLY 时，放宽下载器的 SafeSession 以允许私有 IP。"""
    if self.egress_context.scope == EgressScope.PRIVATE_ONLY:
        session = getattr(downloader, "session", None)
        if session and hasattr(session, "allow_private_ips"):
            session.allow_private_ips = True  # 允许访问本地 Ollama/SearXNG
```

### 5.10.4 专用下载器架构

所有专用下载器继承自 `BaseDownloader`，提供统一的 `download_with_result()` 接口：

```python
class BaseDownloader:
    def download_with_result(self, url, content_type) -> DownloadResult:
        """返回 DownloadResult(is_success, content, skip_reason)"""
```

| 下载器 | 来源 | 特殊能力 |
|--------|------|----------|
| `ArxivDownloader` | arxiv.org | 调用 ArXiv API 获取 LaTeX/PDF |
| `PubMedDownloader` | pubmed.ncbi.nlm.nih.gov | 调用 E-utilities API 获取摘要 + PMC 全文 |
| `SemanticScholarDownloader` | semanticscholar.org | 调用 S2 API 获取论文元数据 |
| `BioRxivDownloader` | biorxiv.org | 调用 bioRxiv API 获取预印本 |
| `DirectPDFDownloader` | 任意 .pdf URL | 直接下载 PDF 并提取文本 |
| `AutoHTMLDownloader` | 任意网页 | Playwright 渲染 + justext 提取 |

### 5.10.5 Playwright HTML 渲染

`AutoHTMLDownloader` 通过 Crawl4AI 集成 Playwright 实现 JavaScript 渲染：

```python
class AutoHTMLDownloader:
    def __init__(self, timeout, language, enable_js_rendering=False):
        self.enable_js_rendering = enable_js_rendering

    def download_with_result(self, url, content_type):
        if self.enable_js_rendering:
            # 使用 Playwright 渲染 JavaScript 页面
            return self._fetch_with_playwright(url)
        else:
            # 使用普通 HTTP 请求 + justext 提取
            return self._fetch_with_requests(url)
```

**默认禁用 JS 渲染的原因**：Docker 生产镜像默认不含 Chromium，且基准测试显示 JS 渲染未显著提升研究质量。

### 5.10.6 Mermaid 时序图

```mermaid
sequenceDiagram
    participant Caller as 调用者
    participant CF as ContentFetcher
    participant UC as URLClassifier
    participant SSRF as policy_aware_validate_url
    participant Egress as evaluate_url
    participant D as 下载器

    Caller->>CF: fetch(url)
    CF->>UC: classify(url)
    UC-->>CF: URLType.ARXIV

    alt URLType.INVALID
        CF-->>Caller: error("Invalid URL")
    end

    CF->>SSRF: validate(url)
    alt SSRF 失败
        CF-->>Caller: error("URL failed security validation")
    end

    CF->>Egress: evaluate(url)
    alt scope 不兼容
        CF-->>Caller: error("URL refused by egress policy")
    end

    CF->>CF: _get_downloader(ARXIV)
    alt 下载器未缓存
        CF->>D: ArxivDownloader(timeout)
        CF->>CF: _apply_egress_policy_to_downloader
        CF->>CF: cache downloader
    end

    CF->>D: download_with_result(url, TEXT)
    D-->>CF: DownloadResult

    alt 专业下载器失败 && 非 HTML/DOI/PDF
        CF->>CF: _get_downloader(HTML)
        CF->>D: html_downloader.download_with_result
        D-->>CF: DownloadResult (HTML 回退)
    end

    CF->>CF: 解码内容 (检测 %PDF magic bytes)
    CF->>CF: 截断到 max_length (500KB)
    CF->>D: get_metadata(url)
    D-->>CF: {title, author, published_date}
    CF-->>Caller: {status: success, content, title, ...}
```

**时序图说明**：该图展示了 `ContentFetcher.fetch()` 的完整调用流程。调用者传入 URL 后，首先由 `URLClassifier` 分类 URL 类型，然后依次执行 SSRF 校验和出口策略校验（两道独立防线）。通过的 URL 被路由到对应的专用下载器（如 `ArxivDownloader`）。若专用下载器失败且 URL 类型允许，系统会回退到通用 HTML 下载器尝试提取。获取原始内容后，系统检测 PDF magic bytes（`%PDF`）并调用相应的文本提取器，最后截断到 500KB 并附加元数据返回给调用者。

### 5.10.7 潜在问题与改进点

1. **下载器缓存生命周期**：`ContentFetcher` 按 `URLType` 缓存下载器，但下载器内部的 HTTP session 可能在长连接场景下过期。
2. **PDF 解码过于严格**：仅接受 UTF-8 和 PDF，其他编码（如 Latin-1、GBK）会失败。
3. **HTML 回退可能掩盖错误**：专业下载器失败时自动回退到 HTML，可能产生不一致的输出格式。

---

## 5.11 引用处理系统

**文件**：`src/local_deep_research/citation_handler.py`、`src/local_deep_research/citation_handlers/`、`src/local_deep_research/text_optimization/citation_formatter.py`

引用处理系统负责分析搜索结果、生成引用、格式化引用样式，是连接搜索与报告生成的关键桥梁。

### 5.11.1 文件结构

```
citation_handler.py                          # CitationHandler 门面 (109 行)
citation_handlers/
├── __init__.py
├── base_citation_handler.py                 # BaseCitationHandler 抽象基类
├── standard_citation_handler.py             # StandardCitationHandler
├── forced_answer_citation_handler.py        # ForcedAnswerCitationHandler
└── precision_extraction_handler.py          # PrecisionExtractionHandler
text_optimization/
├── __init__.py
├── citation_formatter.py                    # CitationFormatter (~50KB, 6 种模式)
└── README.md
```

### 5.11.2 CitationHandler 分析流程

**文件**：`citation_handler.py`（109 行）

```python
class CitationHandler:
    """可配置的引用处理器，委托给具体实现。保持向后兼容。"""

    def __init__(self, llm, handler_type=None, settings_snapshot=None):
        self.llm = llm
        self.settings_snapshot = settings_snapshot or {}

        # 从设置确定处理器类型，默认 "standard"
        if handler_type is None:
            handler_type = unwrap_setting(
                self.settings_snapshot.get("citation.handler_type", "standard")
            )

        self._handler = self._create_handler(handler_type)

        # 向后兼容：暴露内部方法
        self._create_documents = self._handler._create_documents
        self._format_sources = self._handler._format_sources

    def _create_handler(self, handler_type: str):
        """基于类型创建合适的处理器。"""
        handler_map = {
            "standard": StandardCitationHandler,
            "forced": ForcedAnswerCitationHandler,
            "forced_answer": ForcedAnswerCitationHandler,
            "browsecomp": ForcedAnswerCitationHandler,
            "precision": PrecisionExtractionHandler,
            "precision_extraction": PrecisionExtractionHandler,
            "simpleqa": PrecisionExtractionHandler,
        }
        handler_cls = handler_map.get(handler_type.lower(), StandardCitationHandler)
        return handler_cls(self.llm, settings_snapshot=self.settings_snapshot)

    def analyze_initial(self, query, search_results) -> Dict:
        return self._handler.analyze_initial(query, search_results)

    def analyze_followup(self, question, search_results, previous_knowledge, nr_of_links) -> Dict:
        return self._handler.analyze_followup(...)
```

**设计模式**：策略模式（Strategy Pattern）。`CitationHandler` 是上下文，根据配置动态选择具体策略。

### 5.11.3 CitationFormatter 6 种模式详解

**文件**：`text_optimization/citation_formatter.py`（约 50KB）

#### CitationMode 枚举

```python
class CitationMode(Enum):
    NUMBER_HYPERLINKS = "number_hyperlinks"           # [1] 带超链接
    DOMAIN_HYPERLINKS = "domain_hyperlinks"           # [arxiv.org] 带超链接
    DOMAIN_ID_HYPERLINKS = "domain_id_hyperlinks"     # [arxiv.org] 或 [arxiv.org-1]
    DOMAIN_ID_ALWAYS_HYPERLINKS = "domain_id_always_hyperlinks"  # [arxiv.org-1] 总是带 ID
    SOURCE_TAGGED_HYPERLINKS = "source_tagged_hyperlinks"        # [arxiv-7] 源标签
    NO_HYPERLINKS = "no_hyperlinks"                   # [1] 无超链接
```

#### 模式对比表

| 模式 | 显示格式 | 唯一性 | 超链接 | 适用场景 |
|------|----------|--------|--------|----------|
| NUMBER_HYPERLINKS | `[1]` | 全局唯一 | ✅ | 默认模式，最简洁 |
| DOMAIN_HYPERLINKS | `[arxiv.org]` | 域名级别 | ✅ | 快速识别来源 |
| DOMAIN_ID_HYPERLINKS | `[arxiv.org]` 或 `[arxiv.org-1]` | 域名+智能 ID | ✅ | 同域名多来源 |
| DOMAIN_ID_ALWAYS_HYPERLINKS | `[arxiv.org-1]` | 域名+序号 | ✅ | 始终带 ID |
| SOURCE_TAGGED_HYPERLINKS | `[arxiv-7]` | 源标签+全局编号 | ✅ | 保留引用顺序 |
| NO_HYPERLINKS | `[1]` | 全局唯一 | ❌ | 纯文本输出 |

#### SOURCE_TAGGED_HYPERLINKS 详细分析

```python
"""保留全局引用编号，并添加从 URL 推导的短源标签：
- 已知学术来源通过 URLClassifier：arxiv-7, pubmed-3
- 普通域名：nytimes.com-9
- 空/本地 URL：local-N

与 DOMAIN_ID_* 模式不同，后缀是原始引用编号，
因此标签永不冲突且与参考文献顺序匹配：
[1] arxiv + [2] openai + [3] arxiv → arxiv-1, openai-2, arxiv-3
"""
```

#### 核心格式化流程

```python
def format_document_split(self, content: str) -> Tuple[str, str]:
    """格式化引用并返回 (answer, sources_md)。"""

    if self.mode == CitationMode.NO_HYPERLINKS:
        return content, ""

    sources_start = self._find_sources_section(content)
    if sources_start == -1:
        return content, ""

    document_content = content[:sources_start]
    sources_content = content[sources_start:]

    sources = self._parse_sources(sources_content)  # 解析 [N] URL 格式

    if self.mode == CitationMode.NUMBER_HYPERLINKS:
        formatted = self._format_number_hyperlinks(document_content, sources)
    elif self.mode == CitationMode.DOMAIN_HYPERLINKS:
        formatted = self._format_domain_hyperlinks(document_content, sources)
    # ... 其他模式

    return formatted, sources_content
```

#### 引用规范化（citation_normalizer.py）

引用规范化负责将各种格式的引用统一为标准格式：

```python
# 支持的引用格式：
# [1], [1, 2, 3], 【1】, Source 1, source 1
self.citation_pattern = re.compile(r"(?<![\[【])[\[【](\d+)[\]】](?![\]】])")
self.comma_citation_pattern = re.compile(r"[\[【](\d+(?:,\s*\d+)+)[\]】]")
self.source_word_pattern = re.compile(r"\b[Ss]ource\s+(\d+)\b")
```

### 5.11.4 URLClassifier 用于源标签推导

`SOURCE_TAGGED_HYPERLINKS` 模式利用 `URLClassifier` 从 URL 推导短源标签：

```python
def _format_source_tagged_hyperlinks(self, content, sources, collections):
    """使用 URLClassifier 推导源标签。"""
    for num, data in sources.items():
        url = data.get("url", "")
        url_type = URLClassifier.classify(url)

        # 已知学术来源使用短名称
        if url_type == URLType.ARXIV:
            tag = "arxiv"
        elif url_type == URLType.PUBMED:
            tag = "pubmed"
        elif url_type == URLType.SEMANTIC_SCHOLAR:
            tag = "semanticscholar"
        else:
            # 普通域名提取主域名
            tag = urlparse(url).netloc  # e.g., "nytimes.com"

        # 格式：arxiv-7（源标签-全局编号）
        label = f"{tag}-{num}"
```

### 5.11.5 Mermaid 流程图

```mermaid
flowchart TD
    A[CitationHandler 初始化] --> B{handler_type?}
    B -->|standard| C[StandardCitationHandler]
    B -->|forced/browsecomp| D[ForcedAnswerCitationHandler]
    B -->|precision/simpleqa| E[PrecisionExtractionHandler]
    B -->|unknown| C

    C --> F[analyze_initial: 初始分析]
    C --> G[analyze_followup: 跟进分析]

    H[LLM 输出 + Sources 节] --> I[CitationFormatter.format_document_split]
    I --> J{模式?}

    J -->|NO_HYPERLINKS| K[返回原文]
    J -->|NUMBER_HYPERLINKS| L[解析 Sources 节]
    J -->|DOMAIN_HYPERLINKS| L
    J -->|DOMAIN_ID_HYPERLINKS| L
    J -->|SOURCE_TAGGED_HYPERLINKS| L

    L --> M[正则匹配引用: N, N,M, 【N】, Source N]
    M --> N{模式?}

    N -->|NUMBER_HYPERLINKS| O[替换为 href=URL]
    N -->|DOMAIN_HYPERLINKS| P[替换为 domain href=URL]
    N -->|DOMAIN_ID_HYPERLINKS| Q[替换为 domain[-id] href=URL]
    N -->|SOURCE_TAGGED_HYPERLINKS| R[URLClassifier.classify]
    R --> S[推导源标签: arxiv, pubmed, domain]
    S --> T[替换为 tag-N href=URL]

    O --> U[返回 formatted_answer + sources_md]
    P --> U
    Q --> U
    T --> U
```

**流程图说明**：该图展示了引用处理系统的完整流程。左侧是 `CitationHandler` 的策略选择：根据 `handler_type` 配置（standard/forced/precision）实例化不同的处理器。处理器调用 LLM 生成带有 `[N]` 引用标记的输出，末尾附 `## Sources` 节。右侧是 `CitationFormatter` 的格式化流程：首先检测引用模式（支持 `[1]`、`[1, 2, 3]`、`【1】`、`Source 1` 等多种格式），解析 Sources 节获取引用编号到 URL 的映射，然后根据所选模式（NUMBER/DOMAIN/SOURCE_TAGGED）将引用替换为带超链接的格式。SOURCE_TAGGED 模式额外调用 `URLClassifier` 从 URL 推导短源标签（如 arxiv、pubmed），生成 `arxiv-7` 格式的标签。

### 5.11.6 潜在问题与改进点

1. **CitationFormatter 体积过大**：50KB 的单文件难以维护，建议拆分为多个模式策略类。
2. **SOURCE_TAGGED 标签冲突**：当同一域名有多个来源时，`tag-N` 中的 N 是全局编号而非域名内序号，可能导致标签不直观。
3. **引用解析正则脆弱**：`[N]` 格式的正则可能误匹配非引用内容（如数学公式 `[1, 2, 3]`）。

---

## 5.12 报告生成器

**文件**：`src/local_deep_research/report_generator.py`（617 行）

报告生成器是研究的最终输出阶段，将初步研究发现转化为结构化的完整报告。

### 5.12.1 IntegratedReportGenerator 类

```python
class IntegratedReportGenerator:
    def __init__(
        self,
        searches_per_section: int = 2,
        search_system: AdvancedSearchSystem | None = None,
        llm: BaseChatModel | None = None,
        settings_snapshot: Optional[Dict] = None,
    ):
        self.searches_per_section = searches_per_section
        self.max_context_sections = get_setting_from_snapshot(
            "report.max_context_sections", default=3, ...
        )
        self.max_context_chars = get_setting_from_snapshot(
            "report.max_context_chars", default=4000, ...
        )
```

### 5.12.2 generate_report() 三步流程

```python
def generate_report(self, initial_findings, query, progress_callback=None):
    # Step 1: 确定报告结构
    structure = self._determine_report_structure(initial_findings, query)
    # 输出: [{"name": "引言", "subsections": [{"name": "背景", "purpose": "..."}]}]

    # Step 2: 逐节研究与生成
    sections = self._research_and_generate_sections(
        initial_findings, structure, query, progress_callback
    )

    # Step 3: 格式化最终报告
    report = self._format_final_report(sections, structure, query)
    return report
```

### 5.12.3 _determine_report_structure() 结构推导

```python
def _determine_report_structure(self, findings, query):
    prompt = f"""
    分析以下研究内容关于: {query}

    内容摘要:
    {combined_content[:1000]}... [截断]

    确定最适合的报告结构:
    1. 分析内容类型（技术、商业、学术等）
    2. 识别主要主题和逻辑分组
    3. 考虑研究的深度和广度

    以精确格式返回目录结构:
    STRUCTURE
    1. [章节名]
       - [子章节] | [目的]
    2. [章节名]
       - [子章节] | [目的]
    ...
    END_STRUCTURE

    不要包含关于来源、引用、参考文献或方法论的章节。
    """
    response = self.model.invoke(prompt)
    # 解析 STRUCTURE/END_STRUCTURE 块
    return structure
```

**关键约束**：明确指示 LLM 不要生成关于来源/参考文献的章节（这些由系统自动附加）。

### 5.12.4 _research_and_generate_sections() 上下文累积

```python
def _research_and_generate_sections(self, initial_findings, structure, query, ...):
    sections = {}
    accumulated_findings: List[str] = []  # 上下文累积

    for i, section in enumerate(structure, 1):
        for j, subsection in enumerate(section["subsections"], 1):
            # 构建前文上下文（避免重复）
            previous_context = self._build_previous_context(accumulated_findings)

            # 生成搜索查询
            subsection_query = f"""
                研究任务: 为关于 '{query}' 的报告创建 '{subsection['name']}' 子章节内容。
                子章节目的: {subsection['purpose']}
                本报告的其他章节:
                {other_sections_text}

                已写内容（不要重复）:
                {previous_context}

                仅关注与子章节目的特定的信息。
                包括独特细节、具体示例和具体数据。
            """

            # 执行搜索（强制单次迭代）
            strategy.settings_snapshot["search.iterations"] = 1
            subsection_results = self.search_system.analyze_topic(subsection_query)

            # 累积发现
            accumulated_findings.append(
                f"[{section['name']} > {subsection['name']}]\n{generated_content}"
            )
```

#### _build_previous_context() 上下文窗口

```python
def _build_previous_context(self, accumulated_findings):
    """从前 N 个已生成章节构建上下文块。"""
    recent = accumulated_findings[-self.max_context_sections:]  # 默认 3
    previous_context = "\n\n---\n\n".join(recent)

    if len(previous_context) > self.max_context_chars:  # 默认 4000
        previous_context = self._truncate_at_sentence_boundary(
            previous_context, self.max_context_chars
        )

    return f"""
        === 已写内容（不要重复） ===
        {previous_context}
        === 已写内容结束 ===

        重要: 上述内容已写。不要重复这些观点、示例或解释。
        专注于上述未涵盖的新信息。
    """
```

### 5.12.5 _format_final_report() 报告组装

```python
def _format_final_report(self, sections, structure, query):
    # 生成目录
    toc = ["# 目录\n"]
    for i, section in enumerate(structure, 1):
        toc.append(f"{i}. **{section['name']}**")
        for j, subsection in enumerate(section["subsections"], 1):
            toc.append(f"   {i}.{j} {subsection['name']} | _{subsection['purpose']}_")

    # 添加研究摘要
    report_parts = ["\n".join(toc), "", "# 研究摘要", ...]

    # 添加各章节
    for section in structure:
        report_parts.append(sections[section["name"]])

    # 附加 Sources 节
    formatted_links = utilities.search_utilities.format_links_to_markdown(
        all_links=self.search_system.all_links_of_system
    )
    final_report_content += "\n\n## Sources\n\n" + formatted_links

    return {"content": final_report_content, "metadata": {...}}
```

**注意**：Sources 节在此处附加是为了内存消费者（MCP `generate_report`、程序化 API）获得完整的组装块。DB 保存端（`research_service.py`）在持久化前通过 `format_document_split` 剥离 Sources 节，保证 `report_content` 仅包含答案。

### 5.12.6 Mermaid 流程图

```mermaid
flowchart TD
    A[generate_report 入口] --> B[_determine_report_structure]
    B --> C[LLM 分析初始发现]
    C --> D[解析 STRUCTURE/END_STRUCTURE]
    D --> E[生成章节结构列表]

    E --> F{遍历每个章节}
    F --> G{遍历每个子章节}
    G --> H[_build_previous_context]
    H --> I[取最近 N 个章节]
    I --> J[截断到 max_context_chars]
    J --> K[添加 不要重复 指令]

    K --> L[生成 subsection_query]
    L --> M[强制 search.iterations = 1]
    M --> N[search_system.analyze_topic]
    N --> O[获取 current_knowledge]
    O --> P[追加到 accumulated_findings]
    P --> G

    G -->|完成| Q[_format_final_report]
    Q --> R[生成目录 TOC]
    R --> S[添加研究摘要]
    S --> T[添加各章节内容]
    T --> U[附加 Sources 节]
    U --> V[返回 content + metadata]
```

**流程图说明**：该图展示了 `IntegratedReportGenerator.generate_report()` 的完整流程。首先通过 LLM 分析初始研究发现，推导出最优报告结构（章节和子章节）。然后进入双层循环：外层遍历章节，内层遍历子章节。每个子章节生成时，先构建前文上下文（取最近 3 个已生成章节，截断到 4000 字符，添加明确的"不要重复"指令），然后生成针对性的搜索查询，执行单次搜索获取内容，并累积到上下文中供后续章节参考。最后格式化最终报告：生成目录、添加研究摘要、拼接各章节、附加 Sources 节。

### 5.12.7 潜在问题与改进点

1. **结构推导依赖 LLM 稳定性**：LLM 可能生成不规范的 STRUCTURE 块，导致解析失败。
2. **上下文截断可能丢失关键信息**：4000 字符限制可能不足以覆盖复杂主题。
3. **迭代次数硬编码**：`search.iterations = 1` 对所有子章节生效，无法根据复杂度调整。

---

## 5.13 文本优化与处理

**目录**：`src/local_deep_research/text_processing/`、`src/local_deep_research/text_optimization/`

### 5.13.1 文件结构

```
text_processing/
├── __init__.py
└── text_cleaner.py              # 文本清洗（remove_surrogates 等）

text_optimization/
├── __init__.py
├── citation_formatter.py        # CitationFormatter（~50KB）
└── README.md
```

### 5.13.2 文本清洗

**文件**：`text_processing/text_cleaner.py`

文本清洗模块处理从 PDF 提取时产生的畸形 Unicode：

```python
def remove_surrogates(text: str) -> str:
    """移除代理对（surrogate pairs）。
    PDF 提取工具可能产生孤立的代理字符（U+D800-U+DFFF），
    这些字符在 UTF-8 编码中无效，会导致数据库插入失败。
    """
    return text.encode("utf-8", errors="replace").decode("utf-8")
```

**典型问题**：`pdfplumber` 等工具在提取 PDF 时可能将某些字形编码为 Unicode 代理区码点（surrogates），这些码点在 UTF-8 中无法表示，导致 `UnicodeEncodeError`。`remove_surrogates` 用 `errors="replace"` 将无法编码的字符替换为 `�`。

### 5.13.3 CitationFormatter 50KB 详细分析

**文件**：`text_optimization/citation_formatter.py`（约 50KB）

这是项目中最大的单文件之一，支持 6 种引用格式化模式。

#### 核心数据结构

```python
class CitationFormatter:
    def __init__(self, mode: CitationMode = CitationMode.NUMBER_HYPERLINKS):
        self.mode = mode
        # 负向前瞻/后瞻避免匹配已格式化的引用
        # 同时匹配 Unicode 透镜括号 【】（U+3010, U+3011）
        self.citation_pattern = re.compile(
            r"(?<![\[【])[\[【](\d+)[\]】](?![\]】])"
        )
        self.comma_citation_pattern = re.compile(
            r"[\[【](\d+(?:,\s*\d+)+)[\]】]"
        )
        self.source_word_pattern = re.compile(r"\b[Ss]ource\s+(\d+)\b")
        self.sources_pattern = re.compile(
            r"^\[(\d+(?:,\s*\d+)*)\]\s*(.+?)(?:\n\s*URL:\s*(.+?))?$",
            re.MULTILINE,
        )
```

#### _parse_sources() 解析

```python
def _parse_sources(self, sources_content: str) -> Dict[str, dict]:
    """从 Sources 节解析引用编号到数据的映射。

    支持的格式:
    [1] Title Here
        URL: https://example.com
    [1, 2, 3] Multiple sources
    """
    sources = {}
    for match in self.sources_pattern.finditer(sources_content):
        nums = [n.strip() for n in match.group(1).split(",")]
        title = match.group(2)
        url = match.group(3)
        for num in nums:
            sources[num] = {"title": title, "url": url}
    return sources
```

#### 各模式格式化函数

```python
def _format_number_hyperlinks(self, content, sources):
    """[N] → [<a href="URL">N</a>]"""
    def formatter(num, data):
        url = data.get("url", "")
        if url:
            return f'[<a href="{url}">{num}</a>]'
        return f"[{num}]"
    return self.citation_pattern.sub(
        lambda m: formatter(m.group(1), sources.get(m.group(1), {})),
        content
    )

def _format_domain_hyperlinks(self, content, sources):
    """[N] → [domain name]"""
    def formatter(num, data):
        url = data.get("url", "")
        domain = urlparse(url).netloc if url else num
        if url:
            return f'[<a href="{url}">{domain}</a>]'
        return f"[{domain}]"
    return self.citation_pattern.sub(...)

def _format_domain_id_hyperlinks(self, content, sources):
    """[N] → [domain] 或 [domain-id]（智能 ID）"""
    domain_counts = {}  # 跟踪每个域名出现次数
    def formatter(num, data):
        url = data.get("url", "")
        domain = urlparse(url).netloc if url else num
        domain_counts[domain] = domain_counts.get(domain, 0) + 1
        count = domain_counts[domain]
        if count > 1:
            return f'[<a href="{url}">{domain}-{count}</a>]'
        return f'[<a href="{url}">{domain}</a>]'
```

### 5.13.4 期刊质量标签格式化

`journal_quality/` 目录负责期刊质量标签的格式化：

```python
def format_journal_quality(quality: str) -> str:
    """将期刊质量枚举值格式化为可读字符串。"""
    quality_labels = {
        "high": "高质量",
        "medium": "中等质量",
        "low": "低质量",
        "predatory": "掠夺性期刊",
    }
    return quality_labels.get(quality, "未知")
```

### 5.13.5 潜在问题与改进点

1. **CitationFormatter 单文件过大**：50KB 单文件难以测试和维护，建议拆分为策略类。
2. **正则表达式性能**：大量正则替换在大文档中可能成为性能瓶颈。
3. **URL 解析不完整**：`urlparse` 对某些边缘 URL（如无 scheme 的 URL）可能返回意外结果。

---

## 5.14 错误处理与重试机制

**目录**：`src/local_deep_research/error_handling/`、`src/local_deep_research/exceptions.py`

### 5.14.1 文件结构

```
error_handling/
├── __init__.py
├── error_reporter.py            # 错误报告器
├── openai_compat_errors.py      # OpenAI 兼容错误重写
└── report_generator.py          # 错误报告生成

exceptions.py                    # 自定义异常层次结构
```

### 5.14.2 自定义异常层次结构

**文件**：`exceptions.py`

```python
# 基础异常
class LocalDeepResearchError(Exception):
    """所有 LDR 异常的基类。"""

# 研究相关
class ResearchError(LocalDeepResearchError):
    """研究过程中的错误。"""

class DuplicateResearchError(ResearchError):
    """重复研究错误。"""

class SystemAtCapacityError(ResearchError):
    """系统达到容量上限。"""

# 队列相关
class QueueFullError(LocalDeepResearchError):
    """队列已满。"""

# 配置相关
class ConfigurationError(LocalDeepResearchError):
    """配置错误。"""

class InvalidSettingsError(ConfigurationError):
    """无效设置。"""

# 安全相关
class SecurityError(LocalDeepResearchError):
    """安全相关错误。"""

class SSRFBlockedError(SecurityError):
    """SSRF 防护阻止的 URL。"""

class EgressPolicyError(SecurityError):
    """出口策略违规。"""

# 向量存储相关
class VectorStoreError(LocalDeepResearchError):
    """向量存储错误。"""

class IndexCorruptionError(VectorStoreError):
    """索引损坏。"""

class IndexDimensionMismatch(VectorStoreError):
    """维度不匹配。"""

# 文件完整性
class FileIntegrityError(LocalDeepResearchError):
    """文件完整性错误。"""
```

**设计模式**：层次化异常结构。从 `LocalDeepResearchError` 基类派生，按功能域（研究、配置、安全、向量存储）分组。

### 5.14.3 OpenAI 兼容错误处理

**文件**：`error_handling/openai_compat_errors.py`

#### 错误重写动机

当 LM Studio、vLLM、llama.cpp server、OpenRouter 等 OpenAI 兼容端点失败时，底层 `openai.*` / `httpx.*` 异常通常不包含提供商名称、配置的 base URL 或模型信息。此助手遍历异常链找到根 SDK 异常，生成包含上下文信息的友好错误消息。

#### _walk_cause() 异常链遍历

```python
def _walk_cause(exc: BaseException) -> BaseException:
    """遍历 __cause__ / __context__ 找到最深层的非包装异常。

    LangChain 通常将底层 openai.* 异常包装在通用 Exception 或 RuntimeError 中；
    我们需要原始类来分发。带循环检测。
    """
    seen: set[int] = set()
    cur: BaseException | None = exc
    deepest: BaseException = exc
    while cur is not None and id(cur) not in seen:
        seen.add(id(cur))
        deepest = cur
        cur = cur.__cause__ or cur.__context__
    return deepest
```

#### _strip_credentials() 凭证剥离

```python
def _strip_credentials(base_url: str | None) -> str:
    """从 base URL 中移除 userinfo（user:password@）。

    用户有时将 API key 直接嵌入 base URL（如 https://user:key@host/v1）。
    绝不能将其回显到 UI 或日志。
    """
    if not base_url:
        return "<unknown>"
    parsed = urlparse(base_url)
    host = parsed.hostname or ""
    if ":" in host:  # IPv6
        host = f"[{host}]"
    if parsed.port:
        host = f"{host}:{parsed.port}"
    return urlunparse(parsed._replace(netloc=host))
```

#### _dispatch() 错误分发

```python
def _dispatch(root_exc, provider_name, base_url, model):
    """根据异常类型分发到对应的友好消息生成器。"""
    if isinstance(root_exc, openai.AuthenticationError):
        return f"{provider_name} 认证失败: API key 无效或已过期。"
    elif isinstance(root_exc, openai.RateLimitError):
        return f"{provider_name} 速率限制: 请稍后重试。"
    elif isinstance(root_exc, openai.APITimeoutError):
        return f"{provider_name} 请求超时。"
    elif isinstance(root_exc, openai.APIStatusError):
        return f"{provider_name} API 错误 ({root_exc.status_code}): {root_exc.message}"
    elif isinstance(root_exc, httpx.ConnectError):
        return f"{provider_name} 连接失败: 无法连接到 {base_url}。{_docker_hint_if_needed()}"
    # ... 更多类型
```

### 5.14.4 tenacity 重试策略

项目在多个模块使用 `tenacity` 库实现自动重试：

```python
from tenacity import (
    retry,
    stop_after_attempt,
    wait_exponential,
    retry_if_exception_type,
    before_sleep_log,
)

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=2, max=30),
    retry=retry_if_exception_type((openai.RateLimitError, openai.APITimeoutError, httpx.ConnectError)),
    before_sleep=before_sleep_log(logger, logging.WARNING),
)
def call_llm_with_retry(self, prompt):
    return self.model.invoke(prompt)
```

**重试策略**：
- 最多 3 次尝试
- 指数退避：2s → 4s → 8s → ... → 最大 30s
- 仅重试特定异常：速率限制、超时、连接错误
- 重试前记录警告日志

### 5.14.5 速率限制处理（AdaptiveRateLimitTracker）

```python
class AdaptiveRateLimitTracker:
    """自适应速率限制跟踪器。

    基于历史成功率动态调整请求速率。
    当检测到 429 错误时，自动降低请求频率；
    当请求成功时，逐步恢复速率。
    """

    def __init__(self, initial_rate: float = 1.0, min_rate: float = 0.1):
        self.current_rate = initial_rate
        self.min_rate = min_rate
        self.success_count = 0
        self.error_count = 0

    def record_success(self):
        self.success_count += 1
        # 每 10 次成功提升 10% 速率
        if self.success_count % 10 == 0:
            self.current_rate = min(self.current_rate * 1.1, 1.0)

    def record_error(self, status_code: int):
        if status_code == 429:  # Too Many Requests
            self.error_count += 1
            # 遇到 429 立即降低 50% 速率
            self.current_rate = max(self.current_rate * 0.5, self.min_rate)

    def get_delay(self) -> float:
        """获取下次请求前的延迟时间。"""
        return 1.0 / self.current_rate
```

### 5.14.6 Mermaid 状态图

```mermaid
stateDiagram-v2
    [*] --> Request: 发起 LLM 请求

    Request --> Success: 200 OK
    Request --> RateLimit: 429 Too Many Requests
    Request --> Timeout: 408/504 Timeout
    Request --> AuthError: 401/403 Auth Error
    Request --> ServerError: 500/502/503 Server Error
    Request --> ConnectionError: Connection Refused

    state RateLimit {
        [*] --> Wait: 指数退避
        Wait --> Retry: 等待完成
    }

    state Timeout {
        [*] --> Wait: 指数退避
        Wait --> Retry: 等待完成
    }

    state ConnectionError {
        [*] --> CheckDocker: 检查 Docker 网络
        CheckDocker --> Retry: host.docker.internal
    }

    Retry --> Request: 重试（最多 3 次）
    AuthError --> [*]: 不重试，立即失败
    ServerError --> Retry: 重试（502/503 only）

    Success --> [*]: 返回结果
    Retry --> MaxRetries: 达到最大重试次数
    MaxRetries --> ErrorReport: 生成 ErrorReport
    ErrorReport --> [*]: 返回友好错误消息

    note right of AuthError
        认证错误不重试：
        重试不会解决认证问题，
        反而可能触发账户锁定。
    end note

    note right of RateLimit
        指数退避：
        2s → 4s → 8s → ... → 30s
        记录 warning 日志
    end note
```

**状态图说明**：该图展示了 LLM 请求的错误处理状态机。从发起请求开始，可能进入成功或各种错误状态。速率限制（429）和超时（408/504）进入指数退避等待状态，等待完成后重试。认证错误（401/403）立即失败不重试（因为重试无法解决认证问题，反而可能触发账户锁定）。连接错误检查 Docker 网络配置后重试。达到最大重试次数（3 次）后生成 `ErrorReport`，通过 `_walk_cause()` 遍历异常链找到根异常，`_strip_credentials()` 剥离敏感信息，最后通过 `_dispatch()` 生成包含提供商名称和 base URL 的友好错误消息返回给用户。

### 5.14.7 潜在问题与改进点

1. **错误重写可能丢失上下文**：`_walk_cause()` 找到最深层的根异常时，可能丢失中间层的上下文信息。
2. **tenacity 与 asyncio 兼容性**：`tenacity` 原生不支持异步函数，需要 `tenacity.asyncio` 适配。
3. **速率限制器状态不持久**：`AdaptiveRateLimitTracker` 在进程重启后丢失历史状态。

---

## 本章小结

本章深入分析了 Local Deep Research 的六大核心子系统：

1. **LLM 提供商抽象层**：通过四层架构（抽象基类 → OpenAI 兼容基类 → 具体实现 → 自动发现注册表）支持 14 个提供商，采用策略模式和模板方法模式。
2. **嵌入与向量存储**：通过门面模式统一 3 个嵌入提供商和 FAISS 向量存储，强调安全不变量（向量仅存 id，文本永不在向量存储中）。
3. **内容提取与文档加载**：通过 URL 分类器 + 专用下载器的路由架构支持 6 种内容来源，多层 SSRF 防护。
4. **引用处理系统**：策略模式支持 3 种引用处理器和 6 种格式化模式，50KB 的 CitationFormatter 是项目最大单文件。
5. **报告生成器**：三步流程（结构推导 → 逐节研究 → 格式化），上下文累积避免重复。
6. **文本优化与错误处理**：层次化异常结构 + tenacity 重试 + 自适应速率限制。

每个子系统都体现了"fail-closed"安全默认、纵深防御、最小权限等安全设计原则。

---

☕️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../res/promotion.jpg)