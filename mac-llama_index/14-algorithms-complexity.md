# 第 14 章：关键算法与复杂度分析

> 本章对 LlamaIndex 的 10 个核心算法进行伪代码描述、时间/空间复杂度分析，并提供优化建议。

---

## 14.1 文档转换链哈希去重算法

### 14.1.1 伪代码

```
function get_transformation_hash(nodes, transformation):
    nodes_str = ""
    for node in nodes:
        nodes_str += node.get_content(metadata_mode=ALL)
    
    transformation_dict = transformation.to_dict()
    transform_string = remove_unstable_values(str(transformation_dict))
    
    return SHA256(nodes_str + transform_string)
```

### 14.1.2 复杂度分析

| 指标 | 复杂度 | 说明 |
|------|--------|------|
| **时间** | O(n·m) | n = 节点数, m = 平均节点内容长度 |
| **空间** | O(n·m) | 存储 nodes_str |
| **哈希计算** | O(L) | L = 输入字符串长度 |

### 14.1.3 优化建议

1. **增量哈希**: 如果节点未变化，复用之前的哈希
2. **并行计算**: 大批量节点可并行计算内容哈希
3. **内容摘要**: 对超长内容先取摘要再哈希

---

## 14.2 向量检索（相似度排序）

### 14.2.1 伪代码

```
function vector_search(query_embedding, index, top_k, mode):
    if mode == DEFAULT:
        # 余弦相似度
        similarities = []
        for doc_embedding in index:
            sim = cosine_similarity(query_embedding, doc_embedding)
            similarities.append(sim)
        
        # 取 top-k
        top_k_indices = argmax_k(similarities, top_k)
        return [(index[i], similarities[i]) for i in top_k_indices]
    
    elif mode == MMR:
        # Maximal Marginal Relevance
        selected = []
        candidates = set(all_indices)
        
        for _ in range(top_k):
            best_score = -inf
            best_idx = None
            
            for idx in candidates:
                relevance = cosine_similarity(query, doc[idx])
                diversity = max(cosine_similarity(doc[idx], s) for s in selected)
                score = λ * relevance - (1-λ) * diversity
                
                if score > best_score:
                    best_score = score
                    best_idx = idx
            
            selected.append(best_idx)
            candidates.remove(best_idx)
        
        return selected
```

### 14.2.2 复杂度分析

| 指标 | 暴力搜索 | ANN (HNSW) |
|------|----------|------------|
| **时间** | O(N·d) | O(log N · d) |
| **空间** | O(N·d) | O(N·d + N·log N) |
| **构建** | O(1) | O(N·log N · d) |

N = 向量数, d = 维度, λ = MMR 权衡参数

### 14.2.3 优化建议

1. **使用 ANN 索引**: HNSW、IVF、PQ 等
2. **向量量化**: 降低存储和计算成本
3. **混合检索**: 向量 + BM25 混合

---

## 14.3 AutoMerging Retriever（层次合并算法）

### 14.3.1 伪代码

```
function auto_merging_retrieve(query, retriever, merging_ratio):
    # 1. 初始检索（检索更多节点）
    nodes = retriever.retrieve(query, top_k=initial_top_k)
    
    # 2. 构建父子关系树
    #    假设节点按文档结构组织为树
    leaf_nodes = nodes
    parent_map = build_parent_map(all_nodes)
    
    # 3. 自底向上合并
    merged_nodes = []
    for leaf in leaf_nodes:
        parent = parent_map[leaf.id]
        siblings = get_siblings(parent)
        
        # 如果检索到的子节点比例超过阈值
        retrieved_ratio = len(siblings ∩ leaf_nodes) / len(siblings)
        
        if retrieved_ratio >= merging_ratio:
            # 使用父节点替代子节点
            merged_nodes.append(parent)
        else:
            merged_nodes.append(leaf)
    
    return merged_nodes
```

### 14.3.2 复杂度分析

| 指标 | 复杂度 | 说明 |
|------|--------|------|
| **时间** | O(k + h) | k = 初始检索数, h = 树高度 |
| **空间** | O(k) | 存储检索结果和合并结果 |

### 14.3.3 优化建议

1. **预计算父子关系**: 避免每次检索时重建
2. **自适应阈值**: 根据查询复杂度动态调整 merging_ratio
3. **缓存合并结果**: 相同查询结构复用

---

## 14.4 Fusion Retriever（倒数排名融合）

### 14.4.1 伪代码

```
function reciprocal_rank_fusion(retrievers, query, top_k):
    # 1. 生成子查询
    sub_queries = llm.generate_sub_queries(query)
    
    # 2. 并行检索
    all_results = []
    for sub_query in sub_queries:
        for retriever in retrievers:
            results = retriever.retrieve(sub_query)
            all_results.append(results)
    
    # 3. RRF 融合
    scores = {}
    for results in all_results:
        for rank, node in enumerate(results):
            doc_id = node.node.id
            if doc_id not in scores:
                scores[doc_id] = 0
            scores[doc_id] += 1 / (k + rank)  # k = 60 (常数)
    
    # 4. 排序取 top-k
    sorted_docs = sort_by_score(scores, descending=True)
    return sorted_docs[:top_k]
```

### 14.4.2 复杂度分析

| 指标 | 复杂度 | 说明 |
|------|--------|------|
| **时间** | O(q·r·k + D·log D) | q = 子查询数, r = 检索器数, k = 每次检索数, D = 唯一文档数 |
| **空间** | O(q·r·k) | 存储所有检索结果 |

### 14.4.3 优化建议

1. **并行执行**: 子查询和检索器并行
2. **提前终止**: 分数足够高时停止
3. **加权融合**: 不同检索器赋予不同权重

---

## 14.5 Tree Index 构建与查询

### 14.5.1 伪代码

```
function build_tree_index(nodes, num_children, llm):
    # 自底向上构建
    current_level = nodes
    
    while len(current_level) > 1:
        next_level = []
        
        # 分组
        groups = chunk(current_level, num_children)
        
        for group in groups:
            # 拼接组内文本
            group_text = concatenate(group)
            
            # LLM 生成摘要
            summary = llm.complete(f"Summarize: {group_text}")
            
            # 创建摘要节点
            summary_node = TextNode(text=summary)
            next_level.append(summary_node)
            
            # 记录父子关系
            for child in group:
                add_parent_child(summary_node, child)
        
        current_level = next_level
    
    return current_level[0]  # 根节点


function query_tree_index(root, query, mode):
    if mode == SELECT_LEAF:
        # 从根开始，每层选择最相关的子节点
        current = root
        while has_children(current):
            children = get_children(current)
            current = select_most_relevant(children, query)
        return [current]
    
    elif mode == ALL_LEAF:
        # 收集所有叶子节点
        return collect_all_leaves(root)
```

### 14.5.2 复杂度分析

| 指标 | 构建 | 查询 (SELECT_LEAF) |
|------|------|---------------------|
| **时间** | O(n·c·L) | O(h·c·d) |
| **空间** | O(n) | O(1) |

n = 原始节点数, c = num_children, L = 平均文本长度, h = 树高度, d = 嵌入维度

### 14.5.3 优化建议

1. **并行摘要**: 同层节点并行生成摘要
2. **缓存嵌入**: 避免重复计算
3. **自适应深度**: 根据文档数量动态调整

---

## 14.6 KeywordTable 提取（RAKE 变体）

### 14.6.1 伪代码

```
function extract_keywords_rake(text, stopwords):
    # 1. 分句
    sentences = split_sentences(text)
    
    # 2. 提取候选关键词
    candidates = []
    for sentence in sentences:
        words = tokenize(sentence)
        phrase = []
        for word in words:
            if word in stopwords:
                if phrase:
                    candidates.append(join(phrase))
                    phrase = []
            else:
                phrase.append(word)
        if phrase:
            candidates.append(join(phrase))
    
    # 3. 计算词频和共现
    word_freq = Counter()
    word_degree = Counter()
    
    for phrase in candidates:
        words = phrase.split()
        degree = len(words) - 1
        for word in words:
            word_freq[word] += 1
            word_degree[word] += degree
    
    # 4. 计算分数
    word_score = {}
    for word in word_freq:
        word_score[word] = word_degree[word] / word_freq[word]
    
    # 5. 关键词评分
    keyword_scores = {}
    for phrase in candidates:
        score = sum(word_score[w] for w in phrase.split())
        keyword_scores[phrase] = score
    
    return sort_by_score(keyword_scores)
```

### 14.6.2 复杂度分析

| 指标 | 复杂度 | 说明 |
|------|--------|------|
| **时间** | O(n + m) | n = 文本长度, m = 候选关键词数 |
| **空间** | O(m) | 存储候选关键词和分数 |

### 14.6.3 优化建议

1. **预计算停用词**: 使用集合加速查找
2. **并行处理**: 多文档并行提取
3. **TF-IDF 增强**: 结合全局词频

---

## 14.7 KnowledgeGraph 三元组抽取与检索

### 14.7.1 伪代码

```
function extract_triplets(text, llm):
    # LLM 抽取三元组
    prompt = f"Extract (subject, predicate, object) triplets from: {text}"
    response = llm.complete(prompt)
    
    triplets = parse_triplets(response.text)
    return triplets


function query_knowledge_graph(query, graph_store, mode):
    if mode == DEFAULT:
        # 关键词匹配
        keywords = extract_keywords(query)
        results = []
        for keyword in keywords:
            triplets = graph_store.get_triplets(keyword)
            results.extend(triplets)
        return results
    
    elif mode == VECTOR:
        # 向量检索
        query_embedding = embed(query)
        return graph_store.vector_search(query_embedding)
```

### 14.7.2 复杂度分析

| 指标 | 抽取 | 查询 (关键词) | 查询 (向量) |
|------|------|--------------|-------------|
| **时间** | O(L) | O(k·t) | O(log N) |
| **空间** | O(T) | O(T) | O(T) |

L = 文本长度, k = 关键词数, t = 每个关键词的三元组数, T = 总三元组数

### 14.7.3 优化建议

1. **批量抽取**: 多文档并行 LLM 调用
2. **图索引**: 为实体和关系建立索引
3. **混合检索**: 关键词 + 向量 + 图遍历

---

## 14.8 SubQuestionQueryEngine（问题分解与合并）

### 14.8.1 伪代码

```
function sub_question_query(query, tools, llm):
    # 1. 问题分解
    sub_questions = llm.decompose_query(query, tools)
    # e.g., "Compare A and B" → ["What is A?", "What is B?", "Compare them"]
    
    # 2. 并行回答子问题
    answers = []
    for sub_q in parallel:
        tool = select_tool(sub_q, tools)
        answer = tool.query(sub_q)
        answers.append(answer)
    
    # 3. 合并答案
    final_answer = llm.synthesize(query, sub_questions, answers)
    return final_answer
```

### 14.8.2 复杂度分析

| 指标 | 复杂度 | 说明 |
|------|--------|------|
| **时间** | O(d + k·q + s) | d = 分解时间, k = 子问题数, q = 单个查询时间, s = 合成时间 |
| **空间** | O(k) | 存储子问题和答案 |

### 14.8.3 优化建议

1. **并行执行**: 子问题并行查询
2. **缓存子问题**: 相同子问题复用答案
3. **自适应分解**: 根据复杂度决定是否分解

---

## 14.9 ReAct Agent 推理循环

### 14.9.1 伪代码

```
function react_agent_run(query, tools, llm, max_iterations):
    memory = [UserMessage(query)]
    reasoning_history = []
    
    for iteration in range(max_iterations):
        # 1. 格式化历史
        formatted = format_react_prompt(tools, memory, reasoning_history)
        
        # 2. LLM 推理
        response = llm.chat(formatted)
        
        # 3. 解析输出
        action, content = parse_react_output(response)
        
        if action == "Finish":
            return content
        
        elif action == "Action":
            # 4. 执行工具
            tool_result = execute_tool(content.tool_name, content.input)
            
            # 5. 记录推理步骤
            reasoning_history.append({
                "thought": content.thought,
                "action": content.tool_name,
                "observation": tool_result
            })
            
            memory.append(AssistantMessage(content.thought))
            memory.append(ToolMessage(tool_result))
    
    # 达到最大迭代
    return early_stop(memory, llm)
```

### 14.9.2 复杂度分析

| 指标 | 复杂度 | 说明 |
|------|--------|------|
| **时间** | O(n·(L + T)) | n = 迭代次数, L = LLM 调用时间, T = 工具执行时间 |
| **空间** | O(n·M) | M = 平均消息长度 |

### 14.9.3 优化建议

1. **并行工具调用**: 无依赖的工具并行执行
2. **提前终止**: 置信度足够高时停止
3. **缓存 LLM 响应**: 相同输入复用
4. **自适应 max_iterations**: 根据任务复杂度调整

---

## 14.10 Workflow 事件循环调度

### 14.10.1 伪代码

```
function workflow_run(start_event, steps):
    context = create_context()
    event_queue = [start_event]
    running_steps = set()
    
    while event_queue or running_steps:
        # 1. 分发事件到匹配的步骤
        for event in event_queue:
            matched_steps = find_steps_for_event(event.type)
            for step in matched_steps:
                task = asyncio.create_task(step.handler(event, context))
                running_steps.add(task)
        
        event_queue.clear()
        
        # 2. 等待步骤完成
        done, running_steps = await asyncio.wait(running_steps, return_when=FIRST_COMPLETED)
        
        # 3. 收集新事件
        for task in done:
            new_events = task.result()
            event_queue.extend(new_events)
            
            # 检查是否 StopEvent
            if any(isinstance(e, StopEvent) for e in new_events):
                return get_stop_event_result(new_events)
    
    return None
```

### 14.10.2 复杂度分析

| 指标 | 复杂度 | 说明 |
|------|--------|------|
| **时间** | O(S·E) | S = 步骤数, E = 事件数 |
| **空间** | O(S + E) | 存储步骤和事件队列 |

### 14.10.3 优化建议

1. **优先级队列**: 重要事件优先处理
2. **批量分发**: 相同类型事件批量处理
3. **超时控制**: 步骤级超时

---

## 14.11 算法复杂度总结

| 算法 | 时间复杂度 | 空间复杂度 | 瓶颈 |
|------|-----------|-----------|------|
| **转换哈希去重** | O(n·m) | O(n·m) | 大文档内容拼接 |
| **向量检索 (ANN)** | O(log N · d) | O(N·d) | 大规模向量 |
| **AutoMerging** | O(k + h) | O(k) | 父子关系构建 |
| **RRF 融合** | O(q·r·k + D·log D) | O(q·r·k) | 多查询多检索器 |
| **Tree Index 构建** | O(n·c·L) | O(n) | LLM 摘要调用 |
| **Tree Index 查询** | O(h·c·d) | O(1) | 树高度 |
| **RAKE 关键词提取** | O(n + m) | O(m) | 停用词查找 |
| **KG 三元组抽取** | O(L) | O(T) | LLM 调用 |
| **子问题分解** | O(d + k·q + s) | O(k) | 子问题数量 |
| **ReAct Agent** | O(n·(L + T)) | O(n·M) | 迭代次数 |
| **Workflow 调度** | O(S·E) | O(S + E) | 事件数量 |

---

## 14.12 优化建议总览

### 14.12.1 计算优化

1. **并行化**: 批量 Embedding、并行工具调用、并行子查询
2. **缓存**: Embedding 缓存、LLM 响应缓存、转换结果缓存
3. **量化**: 向量量化（PQ/SQ）、模型量化（INT8/INT4）
4. **索引**: ANN 索引（HNSW/IVF）、倒排索引、图索引

### 14.12.2 存储优化

1. **分层存储**: 热数据内存、温数据 SSD、冷数据对象存储
2. **压缩**: 向量压缩、文本压缩
3. **分区**: 按时间/文档/用户分区

### 14.12.3 架构优化

1. **异步优先**: 所有 IO 操作异步化
2. **流式处理**: 边处理边输出
3. **分布式**: 分布式摄入、分布式检索

---

## 14.13 小结

本章对 LlamaIndex 的 10 个核心算法进行了详细分析：

1. **哈希去重**: O(n·m) 时间，适用于转换链缓存
2. **向量检索**: ANN 索引将复杂度从 O(N) 降至 O(log N)
3. **AutoMerging**: 层次合并提升上下文质量
4. **RRF 融合**: 多源检索结果融合
5. **Tree Index**: 自底向上聚类，适合长文档
6. **RAKE**: 快速关键词提取
7. **KG 抽取**: LLM 驱动的三元组抽取
8. **子问题分解**: 复杂问题分解为简单子问题
9. **ReAct Agent**: 多步推理循环
10. **Workflow 调度**: 事件驱动的工作流引擎

在下一章（最后一章），我们将描述测试策略和主要测试用例。

---

☕️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../res/promotion.jpg)