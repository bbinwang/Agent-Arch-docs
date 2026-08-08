# 05-07 — rag-starterkit/pipeline.py 代码讲解

> **文件**: `packages/rag-starterkit/src/rag_starterkit/pipeline.py`
> **行数**: ~93 行
> **职责**: RAG 流水线编排
> **预估字数**: ~8,000 字

---

## 1. 文件概览

`Pipeline` 类是 RAG 系统的编排器，管理索引（prepare）和检索（run）两个阶段。它协调 Chunker、Embedder、VectorDB、LLMFilter 四个组件。

---

## 2. 类定义与初始化

```python
class Pipeline:
    def __init__(
        self,
        qdrant_client: AsyncQdrantClient,
        qdrant_collection_name: str,
        rrf_constant: int = 60,
        sparse_embeddings_only: bool = False,
        parsing_kwargs: dict[str, Any] | None = None,
        cache_directory: str | None = None,
        openai_api_key: str | None = None,
        openai_emebdding_model: str | None = None,
        fastembed_model: str | None = None,
        openai_llm_model: str | None = None,
    ):
```

**参数**:
- `qdrant_client`: Qdrant 异步客户端
- `qdrant_collection_name`: 集合名称
- `rrf_constant`: RRF 平滑常数（默认 60）
- `sparse_embeddings_only`: 是否仅使用稀疏嵌入
- `parsing_kwargs`: 实时解析参数
- `cache_directory`: 缓存目录路径
- `openai_api_key`: OpenAI API Key
- `openai_emebdding_model`: 嵌入模型名称
- `fastembed_model`: 稀疏嵌入模型名称
- `openai_llm_model`: LLM 模型名称

**初始化逻辑**:
1. 选择解析策略（实时 vs 缓存）
2. 初始化 Chunker、Embedder、VectorDB、LLMFilter
3. 设置初始状态 `is_ready = False`

---

## 3. prepare() — 索引阶段

```python
async def prepare(self) -> None:
    if not self.is_ready and not (await self.vector_db.check_if_loaded()):
        if inspect.iscoroutinefunction(self.parsing_strategy):
            contents = await self.parsing_strategy(**self.parsing_kwargs)
        else:
            contents = self.parsing_strategy(**self.parsing_kwargs)
        contents = cast(dict[str, str], contents)
        self.file_paths = [key for key in contents]
        chunks = self.chunker.chunk_texts(contents)
        if not self.sparse_only:
            chunks = await self.embedder.embed_chunks(chunks)
        chunks = self.embedder.sparse_embed_chunks(chunks)
        await self.vector_db.configure_collection()
        await self.vector_db.upload(chunks)
    self.is_ready = True
```

**执行流程**:
1. 检查是否已准备或已加载
2. 获取文档内容（异步或同步）
3. 分块
4. 生成嵌入（如果非 sparse_only）
5. 生成稀疏嵌入
6. 配置集合并上传

**幂等性**: 通过 `is_ready` 和 `check_if_loaded()` 确保只执行一次。

---

## 4. run() — 检索阶段

```python
async def run(self, query: str, limit: int = 1) -> tuple[str | None, str | None]:
    if not self.is_ready:
        raise ValueError("Pipeline has not been prepared before running")
    filter_file = await self.filter_llm.generate_filter(query, self.file_paths)
    file_path = (
        filter_file.file_path
        if filter_file is not None and filter_file.confidence > 50
        else None
    )
    results = await self.vector_db.search(query, file_path=file_path, limit=limit)
    context = "\n\n".join([result["content"] for result in results])
    response = await self.filter_llm.generate_response(query, context)
    return response.response if response is not None else None, file_path
```

**执行流程**:
1. 检查是否已准备
2. LLM 文件过滤
3. 混合检索
4. 拼接上下文
5. 生成回答

**返回值**: `(回答字符串, 文件路径)` 元组

---

☕️ 制作不易，请我喝咖啡☕️关注我➕