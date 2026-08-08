# 05-14 — eval_framework/run.py 代码讲解

> **文件**: `packages/eval-framework/src/eval_framework/run.py`
> **行数**: ~101 行
> **职责**: 双框架运行器
> **预估字数**: ~5,000 字

---

## 1. 任务模板

```python
FS_EXPLORER_PROMPT = Template(
    "Search the answer to the following question: '{{question}}' by using one of the PDF files..."
)
FS_EXPLORER_PROMPT_ADVANCED = Template(
    "Search the answer to the following question: '{{question}}' by using one of the text files..."
)
```

---

## 2. run_workflow() — Agent 运行器

```python
async def run_workflow(question: str, advanced: bool = False) -> RunResult:
    if not advanced:
        start_event = InputEvent(task=FS_EXPLORER_PROMPT.render({"question": question}))
    else:
        start_event = InputEvent(task=FS_EXPLORER_PROMPT_ADVANCED.render({"question": question}))
    tool_calls = []
    file_names = []
    start_time = time.time()
    handler = workflow.run(start_event=start_event)
    async for event in handler.stream_events():
        if isinstance(event, ToolCallEvent):
            tool_calls.append(event.tool_name)
            # 收集访问的文件路径
    result = await handler
    end_time = time.time()
    return RunResult(
        time_taken=(end_time - start_time),
        tool_calls=tool_calls,
        error=result.error,
        final_answer=result.final_result,
        file_path=file_names,
    )
```

---

## 3. run_pipeline() — RAG 运行器

```python
PIPELINE = Pipeline(
    qdrant_client=AsyncQdrantClient(location="http://localhost:6333"),
    qdrant_collection_name=QDRANT_COLLECTION,
    cache_directory="tmp/cache",
)

async def run_pipeline(question: str, advanced: bool = False) -> RunResult:
    if advanced:
        PIPELINE.vector_db.collection_name = QDRANT_COLLECTION_ADVANCED
        PIPELINE.vector_db.sparse_only = True
        PIPELINE.sparse_only = True
    await PIPELINE.prepare()
    start_time = time.time()
    try:
        result, file_path = await PIPELINE.run(question)
        error = None
    except Exception as e:
        result, file_path, error = None, None, str(e)
    end_time = time.time()
    return RunResult(...)
```

**⚠️ 注意**: `PIPELINE` 是全局可变实例，`advanced` 模式会修改其状态，可能导致并发问题。

---

☕️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../../res/promotion.jpg)