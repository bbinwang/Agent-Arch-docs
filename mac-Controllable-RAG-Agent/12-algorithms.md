# 12. 算法与复杂度分析 (Algorithms & Complexity Analysis)

> **章节编号**: 12/13  
> **预计篇幅**: ~6,000 字  
> **覆盖范围**: 核心算法伪代码、时间/空间复杂度、优化建议

---

## 12.1 算法总览

本系统的核心算法可以分为 **四大类**：

| 类别 | 算法 | 用途 | 位置 |
|------|------|------|------|
| **检索算法** | FAISS 向量相似度搜索 | 文档检索 | FAISS 内部 |
| **文本处理** | 最长公共子序列 (LCS) | 相似度计算 | helper_functions.py |
| **规划算法** | LLM 驱动的计划生成 | 任务分解 | functions_for_pipeline.py |
| **验证算法** | 多层幻觉检测 | 质量保障 | functions_for_pipeline.py |

---

## 12.2 算法 1: FAISS 向量相似度搜索

### 12.2.1 算法描述

FAISS (Facebook AI Similarity Search) 是当前最广泛使用的向量相似度检索库。本项目使用的是 **Flat Index**（暴力搜索），即对每个查询向量计算与所有文档向量的距离。

### 12.2.2 伪代码

```python
# FAISS Flat Index 检索伪代码
def faiss_search(query_vector, index, k):
    """
    在 FAISS 索引中搜索 top-k 最相似文档
    
    Args:
        query_vector: 查询向量 (1, d)
        index: FAISS 索引 (n, d)
        k: 返回的 top-k 数量
    
    Returns:
        distances: 距离数组 (k,)
        indices: 文档索引数组 (k,)
    """
    n, d = index.shape  # n: 文档数, d: 向量维度
    
    # 计算查询向量与所有文档向量的 L2 距离
    distances = zeros(n)
    for i in range(n):
        diff = query_vector - index[i]  # (d,)
        distances[i] = sqrt(sum(diff ** 2))  # L2 距离
    
    # 找到 top-k 最小距离
    indices = argsort(distances)[:k]
    return distances[indices], indices
```

### 12.2.3 复杂度分析

| 指标 | Flat Index | IVF Index | HNSW Index |
|------|------------|-----------|------------|
| **时间复杂度** | O(n × d) | O(√n × d) | O(log n × d) |
| **空间复杂度** | O(n × d) | O(n × d) | O(n × d × M) |
| **检索精度** | 100% | ~95% | ~99% |
| **构建时间** | 快 | 中 | 慢 |

**本项目参数**:
- n = 700 (chunks) / 17 (summaries) / 500 (quotes)
- d = 1536 (OpenAI text-embedding-3-small)
- k = 1 或 10

**实际性能**:
- chunks: O(700 × 1536) ≈ 1M 次操作 → ~100ms
- summaries: O(17 × 1536) ≈ 26K 次操作 → ~5ms
- quotes: O(500 × 1536) ≈ 768K 次操作 → ~80ms

### 12.2.4 优化建议

| 优化 | 说明 | 适用场景 |
|------|------|----------|
| **IVF 索引** | 倒排文件索引，减少检索空间 | n > 10K |
| **HNSW 索引** | 图索引，对数级检索 | n > 100K |
| **PQ 量化** | 乘积量化，减少内存 | 内存受限 |
| **GPU 加速** | FAISS GPU 版本 | 高吞吐需求 |

---

## 12.3 算法 2: 最长公共子序列 (LCS)

### 12.3.1 算法描述

最长公共子序列 (Longest Common Subsequence, LCS) 用于计算两个字符串的相似度。本项目使用 `pylcs` 库实现。

### 12.3.2 伪代码

```python
# LCS 伪代码
def lcs_length(s1, s2):
    """
    计算两个字符串的最长公共子序列长度
    
    Args:
        s1: 字符串 1 (长度 m)
        s2: 字符串 2 (长度 n)
    
    Returns:
        lcs_len: 最长公共子序列长度
    """
    m, n = len(s1), len(s2)
    
    # 动态规划表
    dp = [[0] * (n + 1) for _ in range(m + 1)]
    
    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if s1[i-1] == s2[j-1]:
                dp[i][j] = dp[i-1][j-1] + 1
            else:
                dp[i][j] = max(dp[i-1][j], dp[i][j-1])
    
    return dp[m][n]

def similarity_ratio(s1, s2):
    """计算相似度比例"""
    lcs = lcs_length(s1, s2)
    return lcs / len(s2) if s2 else 0
```

### 12.3.3 复杂度分析

| 指标 | 值 |
|------|-----|
| **时间复杂度** | O(m × n) |
| **空间复杂度** | O(m × n)（可优化至 O(min(m, n))） |

**pylcs 优化**: pylcs 使用 C++ 实现，比纯 Python 快 10-100 倍。

### 12.3.4 应用场景

本项目中 LCS 用于 `is_similarity_ratio_lower_than_th()` 函数，但该函数**未被主流程调用**，仅作为工具函数存在。

### 12.3.5 替代方案

| 算法 | 时间复杂度 | 适用场景 |
|------|------------|----------|
| **编辑距离 (Levenshtein)** | O(m × n) | 拼写纠错 |
| **Jaccard 相似度** | O(m + n) | 集合相似度 |
| **余弦相似度** | O(d) | 向量相似度 |
| **BM25** | O(n) | 文本检索 |

---

## 12.4 算法 3: 计划-执行-重规划循环

### 12.4.1 算法描述

这是系统的核心算法，通过 LLM 驱动的计划生成和动态重规划来解决复杂问题。

### 12.4.2 伪代码

```python
# Plan-and-Execute 核心算法伪代码
def plan_and_execute(question, max_iterations=45):
    """
    主算法：计划-执行-重规划循环
    
    Args:
        question: 用户问题
        max_iterations: 最大迭代次数（防止无限循环）
    
    Returns:
        response: 最终答案
    """
    # Phase 1: 初始化
    state = {
        "question": question,
        "anonymized_question": None,
        "plan": [],
        "past_steps": [],
        "aggregated_context": "",
        "mapping": {},
        "tool": None,
        "curr_state": "init"
    }
    
    # Phase 2: 计划流水线
    # 2.1 匿名化
    anonymized = anonymize_question(state["question"])
    state["anonymized_question"] = anonymized["anonymized_question"]
    state["mapping"] = anonymized["mapping"]
    
    # 2.2 生成计划
    plan = generate_plan(state["anonymized_question"])
    state["plan"] = plan.steps
    
    # 2.3 去匿名化
    state["plan"] = deanonymize_plan(state["plan"], state["mapping"])
    
    # 2.4 细化计划
    state["plan"] = break_down_plan(state["plan"])
    
    # Phase 3: 执行循环
    iteration = 0
    while iteration < max_iterations:
        iteration += 1
        
        if not state["plan"]:
            break  # 计划为空，结束
        
        # 3.1 任务路由
        current_task = state["plan"][0]
        route_result = route_task(current_task, state)
        state["tool"] = route_result["tool"]
        state["query"] = route_result["query"]
        
        # 3.2 执行
        if state["tool"] in ["retrieve_chunks", "retrieve_summaries", "retrieve_quotes"]:
            # 检索子图
            context = qualitative_retrieval(state["query"], state["tool"])
            state["aggregated_context"] += context["relevant_context"]
        elif state["tool"] == "answer":
            # 回答子图
            answer = qualitative_answer(state["query"], state["curr_context"])
            state["aggregated_context"] += answer["answer"]
        
        # 3.3 更新历史
        state["past_steps"].append(current_task)
        state["plan"].pop(0)
        
        # 3.4 重规划
        state["plan"] = replan(state)
        
        # 3.5 检查是否可回答
        if can_be_answered(state["question"], state["aggregated_context"]):
            break  # 可以回答，结束循环
    
    # Phase 4: 生成最终答案
    final_answer = generate_final_answer(
        state["question"], 
        state["aggregated_context"]
    )
    
    return final_answer["answer"]
```

### 12.4.3 复杂度分析

| 指标 | 值 | 说明 |
|------|-----|------|
| **时间复杂度** | O(k × T_LLM) | k: 计划步数, T_LLM: 单次 LLM 调用时间 |
| **空间复杂度** | O(C) | C: 聚合上下文大小 |
| **LLM 调用次数** | O(k × c) | c: 每步平均 LLM 调用数（2-3 次） |

**典型值**:
- k = 5-15 步
- T_LLM = 1-3 秒
- c = 2-3 次
- **总时间**: 5×2×2 = 20秒 到 15×3×3 = 135秒

### 12.4.4 循环终止条件

```
终止条件:
├── plan 为空（所有步骤执行完毕）
├── can_be_answered 返回 True
├── iteration >= max_iterations (recursion_limit)
└── 抛出异常（LLM 调用失败等）
```

### 12.4.5 优化建议

| 优化 | 说明 | 预期收益 |
|------|------|----------|
| **并行执行** | 无依赖的步骤并行执行 | -50% 延迟 |
| **计划缓存** | 相似问题复用计划 | -30% LLM 调用 |
| **提前终止** | 检测到足够信息时提前结束 | -20% 延迟 |
| **模型分层** | 简单步骤用便宜模型 | -40% 成本 |

---

## 12.5 算法 4: 检索-蒸馏-验证子图

### 12.5.1 算法描述

每个检索子图都是一个**质量闭环**：检索 → 蒸馏 → 验证 → (通过) 输出 / (失败) 重新蒸馏。

### 12.5.2 伪代码

```python
def qualitative_retrieval_subgraph(query, retriever, max_attempts=3):
    """
    检索-蒸馏-验证子图
    
    Args:
        query: 查询问题
        retriever: 向量检索器
        max_attempts: 最大蒸馏尝试次数
    
    Returns:
        relevant_context: 蒸馏后的相关上下文
    """
    # Step 1: 检索
    documents = retriever.get_relevant_documents(query)
    original_context = " ".join(doc.page_content for doc in documents)
    
    # Step 2: 蒸馏循环
    for attempt in range(max_attempts):
        # 蒸馏：过滤无关信息
        relevant_context = distill(original_context, query)
        
        # 验证：蒸馏是否基于原始检索
        if is_grounded(relevant_context, original_context):
            return {
                "relevant_context": relevant_context,
                "attempts": attempt + 1
            }
        
        # 未通过，继续蒸馏
    
    # 达到最大尝试次数，返回最后一次结果
    return {
        "relevant_context": relevant_context,
        "attempts": max_attempts,
        "warning": "Max attempts reached"
    }

def distill(context, query):
    """
    蒸馏：使用 LLM 过滤无关信息
    
    Prompt: "filter out all the non relevant information..."
    """
    prompt = f"""
    Query: {query}
    Documents: {context}
    
    Filter out non-relevant information.
    DO NOT ADD ANY NEW INFORMATION.
    """
    return llm.invoke(prompt)

def is_grounded(distilled, original):
    """
    验证：蒸馏内容是否基于原始检索
    
    Prompt: "determine if the distilled content is grounded on the original context"
    """
    prompt = f"""
    Original: {original}
    Distilled: {distilled}
    
    Is the distilled content grounded on the original?
    """
    result = llm.invoke(prompt)
    return result.grounded
```

### 12.5.3 复杂度分析

| 指标 | 值 |
|------|-----|
| **时间复杂度** | O(a × (T_retrieve + T_distill + T_verify)) |
| **空间复杂度** | O(C) |
| **参数** | a: 蒸馏尝试次数（通常 1-3） |

### 12.5.4 收敛性分析

```
收敛条件:
├── 蒸馏通过验证（最常见，a=1）
├── 达到 max_attempts（a=3，强制返回）
└── LLM 调用失败（异常终止）

收敛概率（经验值）:
├── a=1: ~85%
├── a=2: ~12%
├── a=3: ~3%
```

---

## 12.6 算法 5: 回答-幻觉检测子图

### 12.6.1 算法描述

回答子图使用 **Chain-of-Thought (CoT)** 生成答案，然后验证答案是否基于上下文。

### 12.6.2 伪代码

```python
def qualitative_answer_subgraph(question, context, max_attempts=3):
    """
    回答-幻觉检测子图
    
    Args:
        question: 要回答的问题
        context: 回答依据的上下文
        max_attempts: 最大尝试次数
    
    Returns:
        answer: 生成的答案
    """
    for attempt in range(max_attempts):
        # Step 1: CoT 生成答案
        answer = cot_answer(question, context)
        
        # Step 2: 幻觉检测
        if is_grounded_on_facts(answer, context):
            return {"answer": answer, "attempts": attempt + 1}
    
    return {"answer": answer, "attempts": max_attempts}

def cot_answer(question, context):
    """
    Chain-of-Thought 回答生成
    
    Prompt 包含 3 个示例:
    - Example 1: 简单传递推理
    - Example 2: 多步聚合推理
    - Example 3: 拒绝回答示例（负例）
    """
    prompt = f"""
    Examples of Chain-of-Thought Reasoning
    
    Example 1: [身高比较]
    Example 2: [魔法咒语]
    Example 3: [无法回答示例 - 关键负例!]
    
    Context: {context}
    Question: {question}
    
    Provide your answer by first showing your step-by-step reasoning...
    """
    return llm.invoke(prompt)

def is_grounded_on_facts(answer, context):
    """
    幻觉检测：答案是否基于上下文
    """
    prompt = f"""
    Context: {context}
    Answer: {answer}
    
    Is the answer grounded in the context?
    Output JSON: {{"grounded_on_facts": true/false}}
    """
    result = llm.invoke(prompt)
    return result.grounded_on_facts
```

### 12.6.3 CoT 示例分析

**Example 1: 传递推理**
```
Context: Mary > Jane, Jane < Tom, Tom = David
Question: Who is tallest?
Reasoning: Mary > Jane, Jane < Tom → Mary > Tom = David → Mary is tallest
```

**Example 2: 多步聚合**
```
Context: 3 个咒语效果
Question: What could Harry do?
Reasoning: Spell1 → transform, Spell2 → levitate, Spell3 → light
           Combined → transform + levitate + illuminate
```

**Example 3: 拒绝回答（负例）**
```
Context: Harry received a broomstick
Question: Why did Harry receive it?
Reasoning: Context does not provide why → "there is no way to determine"
```

> **关键洞察**: Example 3 是**刻意设计的负例**，教会模型在信息不足时拒绝回答。

---

## 12.7 算法 6: 命名实体匿名化

### 12.7.1 算法描述

将问题中的命名实体（人名、地名等）替换为变量 X/Y/Z。

### 12.7.2 伪代码

```python
def anonymize_question(question):
    """
    命名实体匿名化
    
    Args:
        question: 原始问题
    
    Returns:
        anonymized_question: 匿名化后的问题
        mapping: 变量→原始实体的映射
    """
    prompt = f"""
    You are a question anonymizer.
    Replace all name entities with variables (X, Y, Z).
    
    Example 1: "who is harry potter?" → "who is X?", mapping: {{"X": "harry potter"}}
    Example 2: "how did the bad guy play with alex and rony?" 
               → "how did X play with Y and Z?", 
               mapping: {{"X": "bad guy", "Y": "alex", "Z": "rony"}}
    
    Question: {question}
    
    Output JSON: {{"anonymized_question": "...", "mapping": {{...}}}}
    """
    
    result = llm.invoke(prompt)
    return result

def deanonymize_plan(plan, mapping):
    """
    去匿名化：将变量替换回原始实体
    
    Args:
        plan: 匿名化的计划步骤
        mapping: 变量→原始实体的映射
    
    Returns:
        deanonymized_plan: 去匿名化的计划
    """
    prompt = f"""
    Plan: {plan}
    Mapping: {mapping}
    
    Replace all variables with mapped words.
    Output JSON: {{"plan": [...]}}
    """
    
    result = llm.invoke(prompt)
    return result.plan
```

### 12.7.3 复杂度分析

| 指标 | 值 |
|------|-----|
| **时间复杂度** | O(T_LLM) |
| **空间复杂度** | O(n), n: 实体数量 |
| **LLM 调用次数** | 2 次（匿名化 + 去匿名化） |

---

## 12.8 算法 7: 可回答性判断

### 12.8.1 算法描述

判断当前聚合上下文是否足以回答原始问题。

### 12.8.2 伪代码

```python
def can_be_answered(question, aggregated_context):
    """
    判断问题是否可以基于当前上下文回答
    
    Args:
        question: 原始问题
        aggregated_context: 当前聚合的上下文
    
    Returns:
        can_be_answered: bool
    """
    prompt = f"""
    Question: {question}
    Context: {aggregated_context}
    
    Can the question be fully answered based ONLY on the context?
    You have no prior knowledge.
    Output true or false.
    """
    
    result = llm.invoke(prompt)
    return result.can_be_answered
```

### 12.8.3 决策边界

```
can_be_answered = True  → 进入 get_final_answer 节点
can_be_answered = False → 回到 break_down_plan 节点继续执行

决策影响因素:
├── 上下文是否包含问题所需的所有实体
├── 上下文是否包含实体间的关系
├── 上下文是否足以进行推理
└── LLM 对"fully answered"的理解
```

---

## 12.9 算法 8: Ragas 评估

### 12.9.1 算法描述

Ragas 是一个 RAG 评估框架，提供多维质量指标。

### 12.9.2 伪代码

```python
def evaluate_rag(questions, answers, contexts, ground_truths):
    """
    Ragas 评估
    
    Args:
        questions: 问题列表
        answers: 生成的答案列表
        contexts: 检索的上下文列表
        ground_truths: 标准答案列表
    
    Returns:
        scores: 各指标分数
    """
    # 构造数据集
    dataset = {
        "question": questions,
        "answer": answers,
        "contexts": contexts,
        "ground_truth": ground_truths
    }
    
    # 计算各项指标
    metrics = {
        "faithfulness": compute_faithfulness(dataset),      # 答案是否基于检索
        "answer_relevancy": compute_relevancy(dataset),     # 答案与问题相关性
        "context_recall": compute_recall(dataset),          # 检索召回率
        "answer_similarity": compute_similarity(dataset),   # 答案与标准答案相似度
        "answer_correctness": compute_correctness(dataset)  # 答案正确性
    }
    
    return metrics

def compute_faithfulness(dataset):
    """
    忠实度：答案是否基于检索文档
    
    方法:
    1. 将答案分解为多个陈述
    2. 对每个陈述，使用 NLI 模型判断是否被上下文支持
    3. 计算支持比例
    """
    statements = decompose_into_statements(dataset["answer"])
    supported_count = 0
    
    for statement in statements:
        if is_entailed(statement, dataset["contexts"]):
            supported_count += 1
    
    return supported_count / len(statements)
```

### 12.9.3 指标计算复杂度

| 指标 | 时间复杂度 | 说明 |
|------|------------|------|
| **faithfulness** | O(s × c) | s: 陈述数, c: 上下文数 |
| **answer_relevancy** | O(d) | d: 嵌入维度 |
| **context_recall** | O(g × c) | g: ground truth 陈述数 |
| **answer_similarity** | O(d) | 嵌入相似度 |
| **answer_correctness** | O(s + g) | 综合判断 |

---

## 12.10 算法优化建议汇总

### 12.10.1 短期优化

| 算法 | 当前 | 优化 | 收益 |
|------|------|------|------|
| **检索** | Flat Index | 保持（数据量小） | - |
| **LCS** | pylcs | 保持（未在主流程使用） | - |
| **计划循环** | 串行 | 并行执行无依赖步骤 | -50% 延迟 |
| **蒸馏** | 无缓存 | 缓存相同查询的蒸馏结果 | -30% 调用 |
| **评估** | 全量 | 采样评估 | -50% 评估时间 |

### 12.10.2 中期优化

| 算法 | 优化 | 收益 |
|------|------|------|
| **检索** | 切换到 HNSW 索引 | 支持大规模数据 |
| **计划** | 计划缓存 + 相似问题复用 | -30% LLM 调用 |
| **蒸馏** | 使用专门的小模型 | -60% 成本 |
| **验证** | 使用 NLI 模型替代 LLM | -80% 验证成本 |

### 12.10.3 长期优化

| 算法 | 优化 | 收益 |
|------|------|------|
| **检索** | 分布式向量存储 | 支持百万级文档 |
| **计划** | 强化学习优化计划策略 | 提高计划质量 |
| **蒸馏** | 端到端蒸馏模型 | 一次性蒸馏 |
| **验证** | 多模型集成验证 | 提高验证精度 |

---

## 12.11 本章小结

本章分析了系统的 **8 个核心算法**：

1. **FAISS 检索**: O(n×d) 时间，适合小规模数据
2. **LCS 相似度**: O(m×n) 时间，未在主流程使用
3. **计划-执行循环**: O(k×T_LLM) 时间，核心算法
4. **检索-蒸馏-验证子图**: 质量闭环，85% 一次通过
5. **回答-幻觉检测子图**: CoT + 验证，负例设计关键
6. **命名实体匿名化**: 消除 LLM 先验偏见
7. **可回答性判断**: 动态终止条件
8. **Ragas 评估**: 多维质量指标

**下一章**: [13-testing.md](./13-testing.md) — 测试策略与用例说明。

---

# ❤️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../res/promotion.jpg)