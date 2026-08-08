# 05-06 — fs_explorer/caching.py 代码讲解

> **文件**: `src/fs_explorer/caching.py`
> **行数**: ~96 行
> **职责**: 磁盘缓存封装和批量缓存加载
> **预估字数**: ~6,000 字

---

## 1. 文件概览

`caching.py` 实现了 `ParsedFileCache` 类（DiskCache 的封装）和 `parse_and_cache()` 异步函数（批量解析并缓存文件）。

---

## 2. 全局常量

```python
CACHING_DIR = Path("tmp/cache")
```

**缓存存储位置**: 项目根目录下的 `tmp/cache/` 文件夹。

---

## 3. ParsedFileCache 类

### 3.1 `__init__`

```python
class ParsedFileCache:
    def __init__(self) -> None:
        self._cache = Cache(directory=str(CACHING_DIR))
        self._is_warmed_up = CACHING_DIR.is_dir()
```

**初始化**: 创建 DiskCache 实例，检查缓存目录是否已存在。

### 3.2 `warmup()`

```python
def warmup(self) -> None:
    if not self._is_warmed_up:
        os.makedirs(CACHING_DIR, exist_ok=True)
        self._is_warmed_up = True
    return None
```

**功能**: 创建缓存目录（如果不存在）。

**幂等性**: 多次调用安全，仅在首次创建目录。

### 3.3 `is_empty` (property)

```python
@property
def is_empty(self) -> bool:
    return len(list(self._cache.iterkeys())) == 0
```

**功能**: 检查缓存是否为空。

### 3.4 `add_file(file_path, content)`

```python
def add_file(self, file_path: str, content: str) -> None:
    resolved_path = str(Path(file_path).resolve())
    self._cache.add(resolved_path, content)
```

**功能**: 将文件内容存入缓存。

**键格式**: 文件绝对路径（`Path.resolve()`）。

### 3.5 `get_file(file_path)`

```python
def get_file(self, file_path: str) -> str | None:
    resolved_path = str(Path(file_path).resolve())
    return cast(str | None, self._cache.get(resolved_path))
```

**功能**: 从缓存获取文件内容。

**返回值**: 缓存命中返回字符串，未命中返回 `None`。

### 3.6 `close()`

```python
def close(self) -> None:
    self._cache.close()
```

**功能**: 关闭缓存连接。

---

## 4. 全局缓存实例

```python
CACHE = ParsedFileCache()
```

**设计**: 全局单例，供所有模块共享。

---

## 5. parse_and_cache() 异步函数

```python
async def parse_and_cache(directory: str, recursive: bool, to_skip: list[str]) -> None:
    logging.basicConfig(...)
    CACHE.warmup()
    dir_path = Path(directory)
    to_skip_resolved = [str((dir_path / path).resolve()) for path in to_skip]

    # 1. 获取文件列表
    if not recursive:
        files = []
        fls = os.listdir(dir_path)
        for fl in fls:
            resolved = str(Path(dir_path / fl).resolve())
            if resolved not in to_skip_resolved:
                files.append(resolved)
    else:
        files = []
        for root, dirs, fls in os.walk(dir_path):
            dirs[:] = [str((Path(root) / d).resolve()) for d in dirs if d not in to_skip_resolved]
            fls[:] = [str((Path(root) / f).resolve()) for f in fls if f not in to_skip_resolved]
            for fl in fls:
                files.append(str((Path(root) / fl).resolve()))

    # 2. 并发解析
    semaphore = asyncio.Semaphore(5)
    parser = LlamaParse(api_key=..., result_type=ResultType.TXT, fast_mode=True)

    async def parse_job(file_path: str) -> None:
        async with semaphore:
            result = cast(JobResult, await parser.aparse(file_path=file_path))
            if result.error is None:
                text = await result.aget_text()
                CACHE.add_file(file_path, text)
            else:
                logging.info(f"Could not parse file {file_path} because of {result.error}")

    await asyncio.gather(*(parse_job(file) for file in files))
```

**功能**: 批量解析目录中的文件并缓存。

**并发控制**: `Semaphore(5)` 限制最大并发度。

---

## 6. 复杂度分析

| 操作 | 时间复杂度 | 空间复杂度 |
|------|-----------|-----------|
| `add_file` | O(m) | O(m) | m = 内容大小 |
| `get_file` | O(1) | O(m) |
| `parse_and_cache` | O(n × API) | O(1) | n = 文件数 |

---

☕️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../../res/promotion.jpg)