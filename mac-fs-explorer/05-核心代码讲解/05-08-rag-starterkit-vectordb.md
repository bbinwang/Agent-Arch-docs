# 05-08 — rag-starterkit/vectordb.py 代码讲解

> **文件**: `packages/rag-starterkit/src/rag_starterkit/vectordb.py`
> **行数**: ~211 行
> **职责**: Qdrant 封装 + RRF 重排序
> **预估字数**: ~8,000 字

---

## 1. 文件概览

`vectordb.py` 包含两个核心类：`SimpleReranker`（RRF 重排序算法）和 `VectorDB`（Qdrant 封装）。

---

## 2. SimpleReranker 类

### 2.1 倒数排名融合（RRF）

```python
def _reciprocal_rank_fusion(self, dense_results, sparse_results):
    rrf_scores = {}
    for rank, result in enumerate(dense_results, start=1):
        content = result["content"]
        rrf_scores[content] = rrf_scores.get(content, 0.0) + 1 / (self.k + rank)
    for rank, result in enumerate(sparse_results, start=1):
        content = result["content"]
        rrf_scores[content] = rrf_scores.get(content, 0.0) + 1 / (self.k + rank)
    return rrf_scores
```

**公式**: `score(d) = Σ 1/(k + rank_i(d))`

**参数 k=60**: 平滑常数，降低高排名项的权重差异。

### 2.2 去重与合并

```python
def _dedupe_and_merge(self, dense_results, sparse_results):
    results_map = {}
    for result in dense_results:
        if result["content"] not in results_map:
            results_map[result["content"]] = result
    for result in sparse_results:
        if result["content"] not in results_map:
            results_map[result["content"]] = result
    return results_map
```

**功能**: 基于 content 去重，保留首次出现的元数据。

### 2.3 重排序

```python
def rerank(self, dense_results, sparse_results, limit=1):
    if len(dense_results) == 0:
        return sparse_results[:limit]
    rrf_scores = self._reciprocal_rank_fusion(dense_results, sparse_results)
    results_map = self._dedupe_and_merge(dense_results, sparse_results)
    reranked_results = []
    for content, result in results_map.items():
        result_copy = result.copy()
        result_copy["score"] = rrf_scores[content]
        reranked_results.append(result_copy)
    reranked_results.sort(key=lambda x: x["score"], reverse=True)
    return reranked_results[:limit]
```

**执行流程**:
1. 计算 RRF 分数
2. 去重合并
3. 按分数排序
4. 返回 top-k

---

## 3. VectorDB 类

### 3.1 configure_collection()

```python
async def configure_collection(self) -> None:
    if await self._client.collection_exists(self.collection_name):
        return None
    else:
        vectors_config = {"dense-text": VectorParams(size=768, distance=Distance.COSINE)}
        sparse_vectors_config = {"sparse-text": SparseVectorParams(index=SparseIndexParams(on_disk=False))}
        if not self.sparse_only:
            await self._client.create_collection(
                collection_name=self.collection_name,
                vectors_config=vectors_config,
                sparse_vectors_config=sparse_vectors_config,
            )
        else:
            await self._client.create_collection(
                collection_name=self.collection_name,
                sparse_vectors_config=sparse_vectors_config,
            )
```

**功能**: 创建 Qdrant 集合（如果不存在）。

**配置**:
- Dense: 768 维，余弦距离
- Sparse: 内存索引

### 3.2 upload()

```python
async def upload(self, data: list[ChunkWithMetadata]) -> None:
    sparse_embeddings = []
    dense_embeddings = []
    payloads = []
    for d in data:
        sparse_embedding = {"sparse-text": SparseVector(indices=..., values=...)}
        sparse_embeddings.append(sparse_embedding)
        if not self.sparse_only:
            dense_embedding = {"dense-text": d["embedding"]}
            dense_embeddings.append(dense_embedding)
        payload = {"content": d["chunk"].text, "file_path": d["file_path"]}
        payloads.append(payload)
    if not self.sparse_only:
        self._client.upload_collection(self.collection_name, vectors=dense_embeddings, payload=payloads, ids=range(...))
    self._client.upload_collection(self.collection_name, vectors=sparse_embeddings, payload=payloads, ids=range(...))
```

**功能**: 上传嵌入向量到 Qdrant。

### 3.3 search()

```python
async def search(self, query: str, file_path: str | None = None, limit: int = 1):
    dense_results = []
    sparse_results = []
    if file_path:
        filt = Filter(must=FieldCondition(key="file_path", match=MatchValue(value=file_path)))
    else:
        filt = None
    if not self.sparse_only:
        dense_embedding = await self.embedder.embed_query(query)
        result_dense = await self._client.query_points(...)
        for point in result_dense.points:
            dense_results.append(SearchResult(...))
    sparse_embedding = self.embedder.sparse_embed_query(query)
    result_sparse = await self._client.query_points(...)
    for point in result_sparse.points:
        sparse_results.append(SearchResult(...))
    return self._reranker.rerank(dense_results, sparse_results, limit)
```

**功能**: 执行混合检索并返回重排序结果。

---

☕️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../../res/promotion.jpg)