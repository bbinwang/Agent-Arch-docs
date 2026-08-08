# 05-09 — rag-starterkit/embed.py 代码讲解

> **文件**: `packages/rag-starterkit/src/rag_starterkit/embed.py`
> **行数**: ~57 行
> **职责**: 稠密和稀疏嵌入生成
> **预估字数**: ~5,000 字

---

## Embedder 类

```python
DEFAULT_EMBEDDING_MODEL = "text-embedding-3-small"
DEFAULT_FASTEMBED_MODEL = "Qdrant/bm25"

class Embedder:
    def __init__(self, api_key: str, openai_model=None, fastembed_model=None):
        self._client = AsyncOpenAI(api_key=api_key)
        self.model = openai_model or DEFAULT_EMBEDDING_MODEL
        self._sparse_embedder = SparseTextEmbedding(
            model_name=(fastembed_model or DEFAULT_FASTEMBED_MODEL),
            cache_dir="tmp/fastembed",
        )

    async def embed_chunks(self, chunks):
        texts = [chunk["chunk"].text for chunk in chunks]
        embeddings = await self._client.embeddings.create(input=texts, model=self.model, dimensions=768)
        for i, embedding in enumerate(embeddings.data):
            chunks[i]["embedding"] = embedding.embedding
        return chunks

    def sparse_embed_chunks(self, chunks):
        texts = [chunk["chunk"].text for chunk in chunks]
        embeddings = list(self._sparse_embedder.embed(texts))
        for i, embedding in enumerate(embeddings):
            chunks[i]["sparse_embedding"] = embedding
        return chunks

    async def embed_query(self, query: str) -> list[float]:
        embeddings = await self._client.embeddings.create(input=query, model=self.model, dimensions=768)
        return embeddings.data[0].embedding

    def sparse_embed_query(self, query: str) -> SparseEmbedding:
        embeddings = list(self._sparse_embedder.query_embed(query=query))
        return embeddings[0]
```

**设计要点**:
- Dense: OpenAI 异步 API，768 维
- Sparse: FastEmbed 本地 BM25，同步执行
- 批处理支持（`embed_chunks` 批量处理）

---

# ❤️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../../res/promotion.jpg)