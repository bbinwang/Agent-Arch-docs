# 06-03 — DiskCache 缓存设计

> **本章内容**: DiskCache 结构、键设计、过期策略
> **预估字数**: ~5,000 字

---

## 1. 存储引擎

**底层**: SQLite（通过 `diskcache` 库）

**优势**:
- 零配置，无需额外服务
- 事务安全
- 持久化存储
- 支持并发读写

---

## 2. 存储位置

| 缓存类型 | 目录 | 用途 |
|---------|------|------|
| 文件解析缓存 | `tmp/cache/` | 存储 LlamaParse 解析结果 |
| FastEmbed 模型缓存 | `tmp/fastembed/` | 存储下载的 BM25 模型 |

---

## 3. 键值设计

### 3.1 文件解析缓存

| 属性 | 值 | 说明 |
|------|---|------|
| 键格式 | `str(Path(file_path).resolve())` | 文件绝对路径 |
| 值格式 | `str` | 解析后的纯文本 |
| 示例 | `/Users/wangbin/projects/fs-explorer/data/testfile.txt` | 绝对路径 |

### 3.2 键设计理由

- **绝对路径唯一**: 避免相对路径歧义
- **resolve() 规范化**: 解析符号链接和 `..`
- **可读性**: 便于调试和查询

---

## 4. 缓存操作

### 4.1 写入

```python
CACHE.add_file(file_path, content)
# 内部调用: self._cache.add(resolved_path, content)
```

### 4.2 读取

```python
content = CACHE.get_file(file_path)
# 返回: str | None
```

### 4.3 检查空缓存

```python
if CACHE.is_empty:
    # 缓存为空，需要预加载
```

---

## 5. 缓存策略

### 5.1 无过期策略

当前实现没有设置 TTL（Time-To-Live），缓存永久有效。

**影响**:
- 文件内容变更后缓存不会自动更新
- 需要手动清除缓存或重新加载

### 5.2 缓存失效

```python
# 手动删除缓存（需要直接操作 DiskCache）
cache = Cache(directory="tmp/cache")
cache.delete(key)
```

---

## 6. 性能分析

| 操作 | 耗时 | 说明 |
|------|------|------|
| 缓存读取 | < 10ms | SQLite 索引查找 |
| 缓存写入 | < 50ms | SQLite 插入 |
| 缓存检查 | < 5ms | 遍历键 |

---

## 7. 改进建议

### 7.1 添加 TTL

```python
# diskcache 支持 TTL
cache.add(key, value, expire=86400)  # 24 小时过期
```

### 7.2 文件变更检测

```python
# 使用文件修改时间作为缓存验证
import os
mtime = os.path.getmtime(file_path)
cache_key = f"{file_path}:{mtime}"
```

### 7.3 缓存大小限制

```python
# diskcache 支持大小限制
cache = Cache(directory="tmp/cache", size_limit=10**9)  # 1GB
```

---

☕️ 制作不易，请我喝咖啡☕️关注我➕