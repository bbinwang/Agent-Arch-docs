# 05-10 — rag-starterkit/chunk.py 代码讲解

> **文件**: `packages/rag-starterkit/src/rag_starterkit/chunk.py`
> **行数**: ~36 行
> **职责**: 文本分块
> **预估字数**: ~4,000 字

---

## Chunker 类

```python
class ChunkWithMetadata(TypedDict):
    chunk: Chunk
    file_path: str
    embedding: list[float]
    sparse_embedding: SparseEmbedding | None

class Chunker:
    def __init__(self) -> None:
        self._chunker = SentenceChunker(
            chunk_overlap=200,  # 10% 重叠
            chunk_size=2048,
        )

    def chunk_texts(self, contents: dict[str, str]) -> list[ChunkWithMetadata]:
        texts = list(contents.values())
        files = list(contents.keys())
        batch_chunks = self._chunker.chunk_batch(texts=texts)
        chunks_w_meta = []
        for i, batch_chunk in enumerate(batch_chunks):
            for chunk in batch_chunk:
                chunks_w_meta.append(
                    ChunkWithMetadata(
                        chunk=chunk,
                        file_path=files[i],
                        embedding=[],
                        sparse_embedding=None,
                    )
                )
        return chunks_w_meta
```

**设计要点**:
- `chunk_size=2048`: 每块最大字符数
- `chunk_overlap=200`: 块间重叠，确保上下文连贯
- `SentenceChunker`: 在句子边界处切分
- `chunk_batch`: 批量处理提升效率

---

☕️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../../res/promotion.jpg)