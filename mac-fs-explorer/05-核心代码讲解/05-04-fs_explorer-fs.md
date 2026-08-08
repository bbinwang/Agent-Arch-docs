# 05-04 — fs_explorer/fs.py 代码讲解

> **文件**: `src/fs_explorer/fs.py`
> **行数**: ~95 行
> **职责**: 文件系统工具函数集
> **预估字数**: ~8,000 字

---

## 1. 文件概览

`fs.py` 提供了 5 个文件系统工具函数，供 Agent 在探索过程中调用。这些函数是 Agent 与文件系统交互的唯一通道。

---

## 2. 导入声明

```python
import os
import re
import glob
from typing import cast
from llama_cloud_services.parse.utils import ResultType
from llama_cloud_services.parse.types import JobResult
from llama_cloud_services import LlamaParse
from .caching import CACHE, CACHING_DIR
```

---

## 3. 函数详解

### 3.1 `describe_dir_content(directory: str) -> str`

```python
def describe_dir_content(directory: str) -> str:
    if not os.path.exists(directory) or not os.path.isdir(directory):
        return f"No such directory: {directory}"
    children = os.listdir(directory)
    if not children:
        return f"Directory {directory} is empty"
    description = f"Content of {directory}\n"
    files = []
    directories = []
    for child in children:
        fullpath = os.path.join(directory, child)
        if os.path.isfile(fullpath):
            files.append(fullpath)
        else:
            directories.append(fullpath)
    description += "FILES:\n- " + "\n- ".join(files)
    if not directories:
        description += "\nThis folder does not have any sub-folders"
    else:
        description += "\nSUBFOLDERS:\n- " + "\n- ".join(directories)
    return description
```

**功能**: 生成目录内容的文本描述，供 Agent 了解当前环境。

**输出示例**:
```
Content of ./data
FILES:
- ./data/dr_a_order.pdf
- ./data/dr_a_complaint.pdf
- ./data/testfile.txt
SUBFOLDERS:
- ./data/benchmark
```

**设计要点**:
- 分离 FILES 和 SUBFOLDERS，结构清晰
- 使用完整路径（`os.path.join`），避免歧义
- 空目录和不存在目录有特殊提示

### 3.2 `read_file(file_path: str) -> str`

```python
def read_file(file_path: str) -> str:
    if not os.path.exists(file_path) or not os.path.isfile(file_path):
        return f"No such file: {file_path}"
    with open(file_path, "r") as f:
        return f.read()
```

**功能**: 读取文本文件内容。

**错误处理**: 文件不存在时返回错误字符串而非抛出异常。

**⚠️ 限制**: 
- 使用系统默认编码，非 UTF-8 文件可能出错
- 大文件会一次性读入内存

### 3.3 `grep_file_content(file_path: str, pattern: str) -> str`

```python
def grep_file_content(file_path: str, pattern: str) -> str:
    if not os.path.exists(file_path) or not os.path.isfile(file_path):
        return f"No such file: {file_path}"
    with open(file_path, "r") as f:
        content = f.read()
    r = re.compile(pattern=pattern, flags=re.MULTILINE)
    matches = r.findall(content)
    if matches:
        return f"MATCHES for {pattern} in {file_path}:\n\n- " + "\n- ".join(matches)
    return "No matches found"
```

**功能**: 正则表达式搜索文件内容。

**设计要点**:
- 使用 `re.MULTILINE` 支持多行匹配
- `findall` 返回所有匹配（非仅首个）
- 无匹配时返回明确提示

### 3.4 `glob_paths(directory: str, pattern: str) -> str`

```python
def glob_paths(directory: str, pattern: str) -> str:
    if not os.path.exists(directory) or not os.path.isdir(directory):
        return f"No such directory: {directory}"
    matches = glob.glob(f"./{directory}/{pattern}")
    if matches:
        return f"MATCHES for {pattern} in {directory}:\n\n- " + "\n- ".join(matches)
    return "No matches found"
```

**功能**: 文件名模式匹配。

**注意**: 路径拼接使用 `./{directory}/{pattern}`，这意味着 directory 是相对路径。

### 3.5 `check_api_key() -> str`

```python
def check_api_key() -> str:
    message = ""
    if os.getenv("LLAMA_CLOUD_API_KEY") is not None:
        message += "LLAMA_CLOUD_API_KEY is set and you can use the 'parse_file' tool"
        if CACHING_DIR.is_dir():
            message += " in all its functionalities"
        return message
    else:
        if CACHING_DIR.is_dir():
            message += "LLAMA_CLOUD_API_KEY is not set and you can use 'parse_file', but you will only have access to cached files. You should try to use the tool nevertheless."
        else:
            message += "LLAMA_CLOUD_API_KEY is not set and you cannot use the 'parse_file' tool"
        return message
```

**功能**: 检查 LlamaParse API Key 配置状态，指导 Agent 是否可以使用 `parse_file` 工具。

**四种状态**:
1. API Key 已设置 + 缓存存在 → 全部功能
2. API Key 已设置 + 缓存不存在 → 可使用
3. API Key 未设置 + 缓存存在 → 仅缓存
4. API Key 未设置 + 缓存不存在 → 不可使用

### 3.6 `parse_file(file_path: str) -> str` (async)

```python
async def parse_file(file_path: str) -> str:
    if not os.path.exists(file_path) or not os.path.isfile(file_path):
        return f"No such file: {file_path}"
    # 1. 检查缓存
    if (content := CACHE.get_file(file_path)) is not None:
        return content
    # 2. 检查 API Key
    if os.getenv("LLAMA_CLOUD_API_KEY") is None:
        return f"Not possible to parse {file_path} because it has not been cached and the necessary credentials (`LLAMA_CLOUD_API_KEY`) are not set in the environment"
    # 3. 调用 LlamaParse
    parser = LlamaParse(
        api_key=cast(str, os.getenv("LLAMA_CLOUD_API_KEY")),
        result_type=ResultType.TXT,
        fast_mode=True,
    )
    result = cast(JobResult, await parser.aparse(file_path=file_path))
    if result.error is None:
        return await result.aget_text()
    else:
        return f"There was an error while parsing the file {file_path}: {result.error} (code: {result.error_code})"
```

**功能**: 解析非结构化文档（PDF/PPTX/DOCX/XLSX）。

**执行流程**:
1. 检查文件存在性
2. 检查缓存（`CACHE.get_file()`）
3. 检查 API Key
4. 调用 LlamaParse API
5. 返回解析结果

**设计要点**:
- 缓存优先，避免重复 API 调用
- `fast_mode=True` 加速解析
- 异步执行，不阻塞事件循环

---

## 4. 复杂度分析

| 函数 | 时间复杂度 | 空间复杂度 |
|------|-----------|-----------|
| `describe_dir_content` | O(n) | O(n) | n = 目录项数 |
| `read_file` | O(m) | O(m) | m = 文件大小 |
| `grep_file_content` | O(m) | O(m) | m = 文件大小 |
| `glob_paths` | O(k) | O(k) | k = 匹配数 |
| `check_api_key` | O(1) | O(1) |
| `parse_file` | O(1) 缓存 / O(API) 网络 | O(m) |

---

☕️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../../res/promotion.jpg)