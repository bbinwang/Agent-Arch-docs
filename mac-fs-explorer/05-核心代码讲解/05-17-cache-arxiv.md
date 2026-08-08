# 05-17 — cache-arxiv 代码讲解

> **文件**: `packages/cache-arxiv/src/cache_arxiv/main.py` + `utils.py`
> **行数**: ~61 行
> **职责**: arXiv 论文缓存工具
> **预估字数**: ~4,000 字

---

## main.py — CLI 入口

```python
app = Typer()

@app.command()
def main(
    directory: Annotated[str | None, Option("-d", "--directory")] = None,
    cache_directory: Annotated[str | None, Option("-c", "--cache-dir")] = None,
    texts_directory: Annotated[str | None, Option("-t", "--texts-dir")] = None,
) -> None:
    cache_texts(directory, cache_directory, texts_directory)
```

**参数**:
- `directory`: 基础目录（默认当前目录）
- `cache_directory`: 缓存子目录（默认 `tmp/cache`）
- `texts_directory`: 文本子目录（默认 `texts`）

---

## utils.py — 缓存逻辑

```python
BASE_DIRECTORY = "."
CACHE_DIRECTORY = "tmp/cache"
TEXTS_DIRECTORY = "texts"

def cache_texts(base_dir=None, cache_directory=None, texts_directory=None) -> None:
    cache = Cache(directory=cache_directory or CACHE_DIRECTORY)
    files_dir = Path((base_dir or BASE_DIRECTORY)) / (texts_directory or TEXTS_DIRECTORY)
    for root, _, files in files_dir.walk():
        for file in files:
            path = root / file
            text = path.read_text()
            cache.add(str(path.resolve()), text)
    return None
```

**功能**: 遍历 `texts/` 目录中的所有文本文件，读取内容并写入 DiskCache。

**键**: 文件绝对路径
**值**: 文件文本内容

---

☕️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../../res/promotion.jpg)