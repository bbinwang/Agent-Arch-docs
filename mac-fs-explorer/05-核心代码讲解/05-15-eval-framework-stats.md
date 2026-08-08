# 05-15 — eval_framework/stats.py 代码讲解

> **文件**: `packages/eval-framework/src/eval_framework/stats.py`
> **行数**: ~213 行
> **职责**: 统计聚合和报告生成
> **预估字数**: ~5,000 字

---

## 1. 数据结构

```python
class TimeAverage(TypedDict):
    fs_explorer: float
    rag: float
    best: FrameworkType

class LLMAverage(TypedDict):
    correctness: float
    relevance: float

class LLMStats(TypedDict):
    fs_explorer: LLMAverage
    rag: LLMAverage
    best_correctness: FrameworkType
    best_relevance: FrameworkType
```

---

## 2. 统计函数

### get_time_average()

```python
def get_time_average(time_stats: list[BestTime]) -> TimeAverage:
    fs_expl = [time["fs_explorer"] for time in time_stats]
    rag = [time["rag"] for time in time_stats]
    fs_expl_mean = mean(fs_expl)
    rag_mean = mean(rag)
    best = "rag" if fs_expl_mean > rag_mean else "fs-explorer"
    return TimeAverage(fs_explorer=fs_expl_mean, rag=rag_mean, best=best)
```

### get_llm_stats()

```python
def get_llm_stats(llm_stats: list[LLMEvaluations]) -> LLMStats:
    # 收集所有非空的正确性和相关性分数
    # 计算平均值
    # 确定优胜者
    ...
```

---

## 3. Markdown 报告生成

```python
def create_markdown_report(eval_stats: EvalStats, num_tasks: int) -> str:
    # 生成包含以下内容的 Markdown 报告：
    # - 总任务数
    # - 时间性能对比表
    # - LLM 评估指标对比表（正确性、相关性）
    # - 综合对比表
    # - 关键发现和总体优胜者
    ...
```

**报告结构**:
1. Summary（总任务数）
2. Time Performance（平均执行时间对比）
3. LLM Evaluation Metrics（正确性、相关性分数对比）
4. Overall Comparison（综合对比表）
5. Key Takeaways（关键发现）
6. Overall Winner（总体优胜者）

---

☕️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../../res/promotion.jpg)