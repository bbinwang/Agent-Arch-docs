# 05-11 — rag-starterkit/parse.py 代码讲解

> **文件**: `packages/rag-starterkit/src/rag_starterkit/parse.py`
> **行数**: ~97 行
> **职责**: 文档解析策略
> **预估字数**: ~5,000 字

---

## 两种解析策略

### 1. parse_directory() — 实时解析

```python
async def parse_directory(directory: str, recursive: bool, to_skip: list[str]) -> dict[str, str]:
    # 1. 遍历目录获取文件列表
    # 2. 并发调用 LlamaParse（Semaphore 5）
    # 3. 收集结果为 {file_path: text}
    files_contents = await asyncio.gather(*(parse_job(file) for file in files))
    return {path: text for path, text in files_contents if el is not None}
```

### 2. contents_from_cache() — 缓存加载

```python
def contents_from_cache(cache_directory: str = "tmp/cache") -> dict[str, str]:
    cache = Cache(directory=cache_directory)
    data = {}
    for key in cache.iterkeys():
        value = cache.get(key)
        if value is not None:
            data[key] = value
    return data
```

**设计**: 策略模式，Pipeline 根据初始化参数选择解析策略。

---

☕️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../../res/promotion.jpg)