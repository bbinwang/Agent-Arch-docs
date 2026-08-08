# 7. API 与接口设计

> **文档编号**：07/14
> **预估字数**：~12,000 字
> **框架**：FastAPI + FastApiMCP

---

## 7.1 API 总览

### 7.1.1 REST API 端点列表

| 方法 | 路径 | 操作 ID | 功能 |
|------|------|---------|------|
| `GET` | `/` | — | 健康检查 |
| `GET` | `/llm` | `rag_generate_answer` | RAG 问答（完整响应） |
| `GET` | `/rag_text_response` | `rag_generate_answer_simple` | RAG 问答（纯文本） |
| `GET` | `/rag_chunks` | `rag_retrieve_chunks` | 仅检索文档片段 |
| `GET` | `/labels` | — | 获取标签列表 |
| `PUT` | `/update` | `rag_update_index` | 更新索引 |

### 7.1.2 MCP 工具列表

通过 FastApiMCP 暴露的 MCP 工具：

| 工具名 | 对应端点 | 功能 |
|--------|----------|------|
| `rag_retrieve_chunks` | `GET /rag_chunks` | 检索相关文档 |
| `rag_generate_answer` | `GET /llm` | 生成完整答案 |
| `rag_generate_answer_simple` | `GET /rag_text_response` | 生成纯文本答案 |
| `rag_update_index` | `PUT /update` | 更新索引 |

---

## 7.2 GET /llm（RAG 问答）

### 7.2.1 接口定义

```python
@api_app.get("/llm", response_model=ResponseModel, operation_id="rag_generate_answer")
async def llmsearch(
    question: str,
    label: str = "",
    llm_bundle: LLMBundle = Depends(get_llm_bundle_cached),
) -> Any:
```

### 7.2.2 请求参数

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `question` | str | ✅ | — | 用户问题 |
| `label` | str | ❌ | `""` | 文档标签过滤 |

### 7.2.3 请求示例

```bash
curl "http://localhost:8000/llm?question=什么是RAG&label=obsidian"
```

### 7.2.4 响应模型

```python
class ResponseModel(BaseModel):
    id: UUID                    # 响应唯一 ID
    question: str               # 用户问题
    response: str               # LLM 生成的答案
    average_score: float        # 检索质量分数
    semantic_search: List[SemanticSearchOutput]  # 来源文档列表
    hyde_response: str          # HyDE 响应（如果启用）
```

### 7.2.5 响应示例

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "question": "什么是RAG",
  "response": "RAG（Retrieval-Augmented Generation）是一种...",
  "average_score": 0.85,
  "semantic_search": [
    {
      "chunk_link": "obsidian://advanced-uri?vault=kb&filepath=笔记.md&heading=介绍",
      "chunk_text": "RAG 是检索增强生成...",
      "metadata": {
        "source": "/path/to/笔记.md",
        "page": -1,
        "score": 0.92,
        "heading": "介绍"
      }
    }
  ],
  "hyde_response": ""
}
```

### 7.2.6 错误响应

```json
// 404 - 标签不存在
{
  "detail": "Label 'unknown' doesn't exist. Use GET /labels to get a list of labels."
}
```

---

## 7.3 GET /rag_text_response（简化问答）

### 7.3.1 接口定义

```python
@api_app.get("/rag_text_response", operation_id="rag_generate_answer_simple")
async def llmsearch_simple(
    question: str,
    label: str = "",
    llm_bundle: LLMBundle = Depends(get_llm_bundle_cached),
) -> str:
```

### 7.3.2 响应

纯文本格式，仅返回生成的答案。

```
RAG（Retrieval-Augmented Generation）是一种结合检索和生成的技术...
```

---

## 7.4 GET /rag_chunks（检索片段）

### 7.4.1 接口定义

```python
@api_app.get("/rag_chunks", operation_id="rag_retrieve_chunks")
async def semanticsearch(question: str):
```

### 7.4.2 请求参数

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `question` | str | ✅ | 用户问题 |

### 7.4.3 响应示例

```json
{
  "sources": [
    {
      "chunk_link": "/path/to/doc.md",
      "chunk_text": "文档片段内容...",
      "metadata": {
        "source": "/path/to/doc.md",
        "score": 0.88
      }
    }
  ]
}
```

---

## 7.5 PUT /update（更新索引）

### 7.5.1 接口定义

```python
@api_app.put("/update", operation_id="rag_update_index")
def update_index(
    llm_bundle: LLMBundle = Depends(get_llm_bundle_cached),
    config: Config = Depends(get_config),
) -> Any:
```

### 7.5.2 响应示例

```json
{
  "original_n_files": 100,
  "updated_n_files": 102,
  "scanned_files": 3,
  "scanned_chunks": 15,
  "changed_files": 1,
  "changed_chunks": 5,
  "deleted_files": 0,
  "deleted_chunks": 0
}
```

### 7.5.3 错误响应

```json
// 500 - Hash 文件不存在
{
  "detail": "Couldn't find hash files. Please re-create the index using current version of the app."
}
```

---

## 7.6 GET /labels（标签列表）

### 7.6.1 接口定义

```python
@api_app.get("/labels")
async def labels() -> List[str]:
```

### 7.6.2 响应示例

```json
["obsidian", "books", "papers"]
```

---

## 7.7 MCP 接口设计

### 7.7.1 MCP 配置

```python
mcp = FastApiMCP(
    api_app,
    name="pyLLMSearch MCP Server",
    description="pyLLMSearch MCP Server",
    describe_all_responses=True,
    describe_full_response_schema=True,
    include_operations=["rag_retrieve_chunks", "rag_generate_answer", "rag_generate_answer_simple", "rag_update_index"],
)
mcp.mount()
```

### 7.7.2 MCP 传输协议

FastApiMCP 使用 **SSE（Server-Sent Events）** 协议：

```
MCP Client ←—SSE—→ FastAPI Server
```

### 7.7.3 MCP 客户端配置

**Cursor/Windsurf 配置**：

```json
{
  "mcpServers": {
    "pyLLMSearch": {
      "url": "http://localhost:8000/mcp"
    }
  }
}
```

### 7.7.4 MCP 工具调用示例

```json
// rag_generate_answer 调用
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "rag_generate_answer",
    "arguments": {
      "question": "什么是混合检索"
    }
  }
}
```

---

## 7.8 依赖注入设计

### 7.8.1 FastAPI Depends

```python
@lru_cache()
def get_config() -> Config:
    return read_config()

@lru_cache()
def get_cached_llm_bundle() -> LLMBundle:
    return get_llm_bundle(get_config())

def get_llm_bundle_cached() -> LLMBundle:
    return get_cached_llm_bundle()
```

**设计 Rationale**：
- `@lru_cache()` 确保配置和 LLM 只加载一次
- `Depends` 注入使测试更容易（可替换依赖）

### 7.8.2 内存管理

```python
def unload_model(llm_bundle: LLMBundle):
    llm_bundle.store = None
    llm_bundle.chain = None
    llm_bundle.reranker = None
    llm_bundle.hyde_chain = None
    llm_bundle.multiquery_chain = None
    get_cached_llm_bundle.cache_clear()
    gc.collect()
    with torch.no_grad():
        torch.cuda.empty_cache()
```

**使用场景**：索引更新后释放 GPU 内存。

---

## 7.9 CORS 配置

```python
origins = ["*"]
api_app.add_middleware(
    CORSMiddleware,
    allow_origins=origins,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

**注意**：默认允许所有来源，生产环境应限制为特定域名。

---

## 7.10 配置加载

### 7.10.1 环境变量

```python
rag_config_file = os.environ["FASTAPI_RAG_CONFIG"]
llm_config_file = os.environ["FASTAPI_LLM_CONFIG"]
```

### 7.10.2 启动方式

```bash
export FASTAPI_RAG_CONFIG=/path/to/doc_config.yaml
export FASTAPI_LLM_CONFIG=/path/to/llm_config.yaml
llmsearchapi
```

或

```bash
python -m llmsearch.api
```

---

## 7.11 错误处理

### 7.11.1 HTTP 异常

```python
if label and (label not in get_config().embeddings.labels):
    raise HTTPException(
        status_code=404,
        detail=f"Label '{label}' doesn't exist. Use GET /labels to get a list of labels.",
    )
```

### 7.11.2 业务异常

```python
try:
    stats = update_embeddings(config, vs)
except EmbeddingsHashNotExistError as exc:
    raise HTTPException(
        status_code=500,
        detail="Couldn't find hash files. Please re-create the index using current version of the app.",
    ) from exc
```

---

## 7.12 本章小结

本章详细分析了 pyLLMSearch 的 API 设计：

1. **REST API**：6 个端点（问答、检索、标签、更新）
2. **MCP 接口**：4 个工具通过 SSE 协议暴露
3. **依赖注入**：FastAPI Depends + lru_cache
4. **错误处理**：HTTPException + 业务异常
5. **CORS**：默认允许所有来源

下一章将分析部署运维与基础设施。

---

# ❤️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../res/promotion.jpg)