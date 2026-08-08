# 05-13 — eval_framework/evaluate.py 代码讲解

> **文件**: `packages/eval-framework/src/eval_framework/evaluate.py`
> **行数**: ~187 行
> **职责**: 评估执行和 LLM-as-Judge
> **预估字数**: ~6,000 字

---

## 1. 文件概览

`evaluate.py` 实现了双框架对比评估的核心逻辑，包括数据集加载、评估循环和 LLM-as-Judge 评分。

---

## 2. 数据结构

### 2.1 TypedDict 定义

```python
class EvalTask(TypedDict):
    question: str
    answer: str
    file: str

class LLMEvaluation(TypedDict):
    relevance: int
    correctness: int
    reason: str

class EvalResult(TypedDict):
    task: EvalTask
    llm_evaluations: LLMEvaluations
    answers: Answers
    time_taken: BestTime
    used_files: FilePath
    tool_calls: list[str]
    has_error: HasError
```

### 2.2 Evaluation 模型

```python
class Evaluation(BaseModel):
    relevance: int = Field(ge=0, le=10)
    correctness: int = Field(ge=0, le=10)
    reason: str
```

---

## 3. LLM-as-Judge

```python
LLM_AS_A_JUDGE_PROMPT = Template(
    "The following question: '{{question}}' has this ground truth answer: '{{ground_truth}}'. "
    "Please evaluate this answer: '{{answer}}' grading its correctness and relevance..."
)
LLM_AS_A_JUDGE_MODEL = "gpt-5.2"

async def llm_as_a_judge(question: str, ground_truth: str, produced_answer: str) -> Evaluation | None:
    content = LLM_AS_A_JUDGE_PROMPT.render({"question": question, "ground_truth": ground_truth, "answer": produced_answer})
    message = EasyInputMessageParam(content=content, role="user")
    client = AsyncOpenAI(api_key=os.getenv("OPENAI_API_KEY"))
    response = await client.responses.parse(
        text_format=Evaluation,
        input=[message],
        reasoning=Reasoning(effort="none"),
        model=LLM_AS_A_JUDGE_MODEL,
    )
    return response.output_parsed
```

**功能**: 使用 GPT-5.2 评估回答的正确性和相关性（0-10 分）。

---

## 4. 评估循环

```python
async def run_evaluation(dataset_file: str, results_file: str = "results.json", advanced: bool = False):
    tasks = get_evaluation_dataset(dataset_file)
    results = []
    try:
        for i, task in enumerate(tasks):
            wf_result = await run_workflow(question=task["question"], advanced=advanced)
            pipeline_result = await run_pipeline(question=task["question"], advanced=advanced)
            # 评估 Agent 回答
            if wf_result["error"] is None and wf_result["final_answer"] is not None:
                wf_evaluation = await llm_as_a_judge(...)
            # 评估 RAG 回答
            if pipeline_result["error"] is None and pipeline_result["final_answer"] is not None:
                pipeline_evaluation = await llm_as_a_judge(...)
            results.append(EvalResult(...))
            await asyncio.sleep(1)  # 限流
    except Exception as e:
        print(f"An error occurred: {e}")
    with open(results_file, "w") as f:
        json.dump(results, f, indent=2)
```

**执行流程**:
1. 加载数据集
2. 遍历每个任务
3. 分别运行 Agent 和 RAG
4. 使用 LLM-as-Judge 评估
5. 保存结果

---

☕️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../../res/promotion.jpg)