# 05-12 — rag-starterkit/llm_filter.py 代码讲解

> **文件**: `packages/rag-starterkit/src/rag_starterkit/llm_filter.py`
> **行数**: ~71 行
> **职责**: LLM 文件过滤和回答生成
> **预估字数**: ~5,000 字

---

## LLMFilter 类

```python
DEFAULT_OPENAI_MODEL = "gpt-4.1"

class FileFilter(BaseModel):
    file_path: str
    confidence: int = Field(ge=0, le=100)

class GroundedResponse(BaseModel):
    response: str

class LLMFilter:
    def __init__(self, api_key: str, model: str | None = None):
        self._client = AsyncOpenAI(api_key=api_key)
        self.model = model or DEFAULT_OPENAI_MODEL

    async def generate_filter(self, query: str, file_paths: list[str]) -> FileFilter | None:
        # 使用 GPT-4.1 从文件列表中筛选最相关的文件
        response = await self._client.responses.parse(
            text_format=FileFilter,
            input=[message],
            model=self.model,
        )
        return response.output_parsed

    async def generate_response(self, query: str, context: str) -> GroundedResponse | None:
        # 使用 GPT-4.1 基于检索上下文生成回答
        response = await self._client.responses.parse(
            text_format=GroundedResponse,
            input=[message],
            model=self.model,
        )
        return response.output_parsed
```

**设计要点**:
- 文件过滤：从文件列表中筛选最相关的一个（confidence > 50 才应用）
- 回答生成：基于检索上下文生成最终回答
- 使用 `responses.parse()` 获取结构化输出

---

☕️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../../res/promotion.jpg)