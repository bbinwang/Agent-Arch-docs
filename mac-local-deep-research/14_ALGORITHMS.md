# 14. 关键算法和复杂逻辑分析

> 本文档深入分析 Local Deep Research 项目中的核心算法，包含完整伪代码、时间/空间复杂度分析以及优化建议。

---

## 目录

- [14.1 迭代深化搜索算法](#141-迭代深化搜索算法)
- [14.2 候选排序与相关性过滤](#142-候选排序与相关性过滤)
- [14.3 向量检索与重排序](#143-向量检索与重排序)
- [14.4 文本分块与嵌入算法](#144-文本分块与嵌入算法)
- [14.5 引用解析与格式化](#145-引用解析与格式化)
- [14.6 限流算法](#146-限流算法)
- [14.7 新闻推荐评分算法](#147-新闻推荐评分算法)
- [14.8 出口策略评估算法](#148-出口策略评估算法)

---

## 14.1 迭代深化搜索算法

### 问题描述

深度研究需要在有限的时间和 token 预算内，通过多轮搜索逐步获取足够信息来回答复杂研究问题。每轮搜索后，系统需要评估当前信息是否充分，决定是继续深入搜索还是生成最终报告。

### 算法思路

SourceBasedSearchStrategy 采用迭代深化搜索（Iterative Deepening Search）思想，将搜索过程建模为多轮循环：

1. **问题分解**：将复杂研究问题分解为多个子问题
2. **并行搜索**：每轮对多个子问题并行执行搜索
3. **结果聚合**：合并所有搜索结果并去重
4. **充分性评估**：使用 LLM 评估当前信息是否充分
5. **动态调整**：根据评估结果调整下一轮搜索策略

### 完整伪代码

```
算法: IterativeDeepeningSearch
输入: query (研究问题), max_iterations (最大迭代数), engines (搜索引擎列表)
输出: report (最终研究报告)

function ITERATIVE_DEEPENING_SEARCH(query, max_iterations, engines):
    // 初始化
    all_results ← ∅
    findings ← ∅
    sub_questions ← DECOMPOSE_QUESTION(query)    // LLM 分解问题

    for iteration ← 1 to max_iterations do
        // 阶段1: 并行搜索所有子问题
        search_tasks ← []
        for each question in sub_questions do
            for each engine in engines do
                task ← ASYNC_SEARCH(engine, question)
                search_tasks.APPEND(task)
            end for
        end for
        batch_results ← AWAIT_ALL(search_tasks)    // 并行执行

        // 阶段2: 结果聚合与去重
        new_results ← DEDUPLICATE(batch_results, all_results)
        all_results ← all_results ∪ new_results

        // 阶段3: 内容获取与摘要
        for each result in new_results do
            content ← FETCH_CONTENT(result.url)
            summary ← SUMMARIZE(content, query)
            findings.APPEND({
                "source": result,
                "summary": summary,
                "relevance": SCORE_RELEVANCE(summary, query)
            })
        end for

        // 阶段4: 充分性评估
        sufficiency ← EVALUATE_SUFFICIENCY(findings, query)
        if sufficiency.score ≥ SUFFICIENCY_THRESHOLD then
            break    // 信息充分，停止搜索
        end if

        // 阶段5: 动态调整搜索策略
        sub_questions ← GENERATE_FOLLOWUPS(findings, query)
        engines ← ADJUST_ENGINES(sufficiency.gaps)
    end for

    // 生成最终报告
    report ← GENERATE_REPORT(findings, query)
    return report

function DECOMPOSE_QUESTION(query):
    // 使用 LLM 将复杂问题分解为 3-5 个子问题
    prompt ← f"将以下研究问题分解为3-5个具体子问题: {query}"
    response ← LLM_GENERATE(prompt)
    return PARSE_SUB_QUESTIONS(response)

function DEDUPLICATE(new_results, existing_results):
    // 基于 URL 和语义相似度去重
    unique ← []
    for result in new_results:
        is_duplicate ← false
        for existing in existing_results:
            if URL_SIMILARITY(result.url, existing.url) > 0.9:
                is_duplicate ← true
                break
            end if
            if SEMANTIC_SIMILARITY(result.snippet, existing.snippet) > 0.85:
                is_duplicate ← true
                break
            end if
        end for
        if NOT is_duplicate:
            unique.APPEND(result)
        end if
    end for
    return unique

function EVALUATE_SUFFICIENCY(findings, query):
    // LLM 评估信息充分性
    context ← FORMAT_FINDINGS(findings)
    prompt ← f"""
    基于以下研究发现，评估信息是否足以回答研究问题。
    研究问题: {query}
    研究发现: {context}
    请给出充分性评分(0-1)和信息缺口描述。
    """
    response ← LLM_GENERATE(prompt)
    return PARSE_SUFFICIENCY(response)
```

### 复杂度分析

| 复杂度类型 | 表达式 | 说明 |
|-----------|--------|------|
| 时间复杂度（最坏） | O(I × Q × E × (R + C + S)) | I=迭代数, Q=子问题数, E=引擎数, R=结果数, C=内容获取, S=摘要生成 |
| 时间复杂度（平均） | O(I × Q × E × R × (1 + αC + βS)) | α=新结果比例, β=摘要比例 |
| 时间复杂度（最好） | O(Q × E × R) | 首轮即满足充分性 |
| 空间复杂度 | O(N × M) | N=总结果数, M=平均内容大小 |

具体分解：

- **搜索阶段**：O(Q × E × R)，每轮对 Q 个子问题在 E 个引擎上搜索，每个引擎返回 R 个结果
- **去重阶段**：O(N²) 最坏情况（N 为累计结果数），使用 LSH 可优化至 O(N)
- **内容获取**：O(N × C)，C 为平均页面获取时间
- **摘要生成**：O(N × S)，S 为 LLM 生成摘要的延迟
- **充分性评估**：O(N × L)，L 为 LLM 评估的 token 消耗

### 优化建议

1. **并行化**：所有子问题的搜索可完全并行，使用 asyncio.gather 实现
2. **早停机制**：设置 token 预算上限，达到上限立即停止
3. **缓存**：对相同查询的搜索结果进行缓存，避免重复请求
4. **增量摘要**：仅对新发现的结果生成摘要，复用已有摘要
5. **自适应迭代**：根据问题复杂度动态调整 max_iterations

### 相关代码位置

```
src/local_deep_research/strategies/source_based_strategy.py
src/local_deep_research/strategies/agent_strategy.py
src/local_deep_research/search_engines/parallel.py
```

---

## 14.2 候选排序与相关性过滤

### 问题描述

多个搜索引擎返回的结果存在质量差异、重复和噪声。需要设计算法对候选结果进行交叉验证、相关性评分和排序，确保最终用于生成报告的信息质量。

### 算法思路

CrossEngineFilter 采用多阶段过滤策略：

1. **多引擎交叉验证**：多个引擎返回的相同结果获得更高置信度
2. **LLM 相关性评分**：使用 LLM 对每个结果进行相关性判断
3. **来源可信度加权**：不同来源类型（学术、新闻、百科）有不同权重
4. **时间衰减**：较新的结果获得更高分数
5. **多样性保证**：确保结果覆盖不同视角

### 完整伪代码

```
算法: CrossEngineFilterAndRank
输入: raw_results (原始搜索结果列表), query (研究问题), config (配置)
输出: ranked_results (排序后的结果列表)

function CROSS_ENGINE_FILTER_AND_RANK(raw_results, query, config):
    // 阶段1: 分组与交叉验证
    url_groups ← GROUP_BY_URL(raw_results)
    validated_results ← []

    for each group in url_groups do
        // 多个引擎返回相同 URL → 高置信度
        confidence ← CALCULATE_CONFIDENCE(group)
        merged ← MERGE_RESULTS(group)
        merged.confidence ← confidence
        validated_results.APPEND(merged)
    end for

    // 阶段2: LLM 相关性评分
    scored_results ← []
    for each result in validated_results do
        relevance_score ← LLM_RELEVANCE_SCORE(result, query)
        if relevance_score ≥ config.min_relevance then
            result.relevance ← relevance_score
            scored_results.APPEND(result)
        end if
    end for

    // 阶段3: 综合评分计算
    for each result in scored_results do
        final_score ← CALCULATE_FINAL_SCORE(result, config)
        result.final_score ← final_score
    end for

    // 阶段4: 多样性重排序
    ranked_results ← DIVERSITY_RERANK(scored_results, config.top_k)

    return ranked_results

function CALCULATE_CONFIDENCE(group):
    // 基于引擎数量和一致性计算置信度
    engine_count ← LENGTH(group.engines)
    snippet_consistency ← SNIPPET_SIMILARITY(group.snippets)

    // 引擎越多、摘要越一致，置信度越高
    confidence ← MIN(1.0, engine_count / 3.0) * 0.6
                + snippet_consistency * 0.4
    return confidence

function CALCULATE_FINAL_SCORE(result, config):
    // 加权综合评分
    score ← 0.0
    score += result.relevance * config.w_relevance        // 相关性权重
    score += result.confidence * config.w_confidence      // 置信度权重
    score += SOURCE_CREDIBILITY(result.source) * config.w_source  // 来源可信度
    score += TIME_DECAY(result.date) * config.w_recency   // 时间衰减
    score += FRESHNESS_BOOST(result.date) * config.w_freshness  // 新鲜度
    return score

function DIVERSITY_RERANK(results, top_k):
    // MMR (Maximal Marginal Relevance) 多样性重排序
    selected ← []
    remaining ← SORT_BY_SCORE(results)

    while LENGTH(selected) < top_k AND remaining ≠ ∅ do
        best ← null
        best_mmr_score ← -∞

        for each candidate in remaining do
            relevance ← candidate.final_score
            max_sim ← MAX_SIMILARITY(candidate, selected)
            mmr ← config.lambda * relevance
                  - (1 - config.lambda) * max_sim

            if mmr > best_mmr_score then
                best ← candidate
                best_mmr_score ← mmr
            end if
        end for

        selected.APPEND(best)
        remaining.REMOVE(best)
    end while

    return selected

function MAX_SIMILARITY(candidate, selected):
    if selected = ∅ then
        return 0
    end if
    max_sim ← 0
    for each s in selected do
        sim ← EMBEDDING_SIMILARITY(candidate, s)
        max_sim ← MAX(max_sim, sim)
    end for
    return max_sim
```

### 复杂度分析

| 复杂度类型 | 表达式 | 说明 |
|-----------|--------|------|
| 时间复杂度（最坏） | O(N² × D + N × L) | N=结果数, D=嵌入维度, L=LLM 调用延迟 |
| 时间复杂度（平均） | O(N log N × D + αN × L) | α=通过初筛的比例 |
| 空间复杂度 | O(N × D) | 存储所有结果的嵌入向量 |

各阶段复杂度：

- **URL 分组**：O(N) 使用哈希表
- **交叉验证**：O(N × G)，G 为平均组大小
- **LLM 评分**：O(N × L)，L 为 LLM 调用延迟（瓶颈）
- **MMR 重排序**：O(K × N × D)，K=top_k，D=嵌入维度

### 优化建议

1. **批量 LLM 评分**：将多个结果打包到一次 LLM 调用中评分
2. **嵌入缓存**：缓存已计算的嵌入向量，避免重复计算
3. **近似最近邻**：使用 FAISS 加速 MMR 中的相似度计算
4. **分层过滤**：先快速过滤明显不相关的结果，再对候选集精细评分
5. **并行评分**：多个结果的 LLM 评分可并行执行

### 相关代码位置

```
src/local_deep_research/advanced_search_system/cross_engine_filter.py
src/local_deep_research/advanced_search_system/ranker.py
src/local_deep_research/advanced_search_system/diversity.py
```

---

## 14.3 向量检索与重排序

### 问题描述

RAG 系统需要将用户查询与本地文档库进行语义匹配，检索最相关的文档片段。FAISS 提供高效的向量相似度搜索，但需要与数据库中的元数据配合使用。

### 算法思路

VectorIndex.search 采用两阶段检索策略：

1. **嵌入查询**：将用户查询转换为向量表示
2. **FAISS 搜索**：在向量索引中搜索最近邻
3. **数据库水合**：从数据库获取完整文档信息
4. **后处理重排序**：基于额外特征进行精排

### 完整伪代码

```
算法: VectorSearchWithReranking
输入: query (用户查询), index (FAISS索引), db (数据库连接), k (返回数量)
输出: results (检索结果列表)

function VECTOR_SEARCH(query, index, db, k):
    // 阶段1: 查询嵌入
    query_embedding ← EMBED_TEXT(query)        // 转换为 768 维向量
    query_embedding ← NORMALIZE(query_embedding) // L2 归一化

    // 阶段2: FAISS 近似搜索
    // 扩大搜索范围以留出重排序空间
    search_k ← k * RERANK_FACTOR              // 通常 RERANK_FACTOR=3
    scores, vector_ids ← index.search(query_embedding, search_k)

    // 阶段3: 数据库水合（获取完整文档信息）
    documents ← []
    for each (score, vid) in ZIP(scores, vector_ids) do
        doc ← db.query("SELECT * FROM documents WHERE vector_id = ?", vid)
        doc.vector_score ← score
        doc.vector_id ← vid
        documents.APPEND(doc)
    end for

    // 阶段4: 后处理重排序
    reranked ← RERANK_DOCUMENTS(documents, query)

    // 阶段5: 返回 Top-K
    return reranked[0:k]

function RERANK_DOCUMENTS(documents, query):
    // 多特征融合重排序
    for each doc in documents do
        // 特征1: 向量相似度（来自 FAISS）
        vector_score ← doc.vector_score

        // 特征2: BM25 文本匹配分数
        bm25_score ← BM25_SCORE(query, doc.content)

        // 特征3: 关键词匹配
        keyword_score ← KEYWORD_MATCH_SCORE(query, doc.content)

        // 特征4: 文档质量分数
        quality_score ← doc.quality_score  // 预计算

        // 特征5: 时间衰减
        time_score ← TIME_DECAY(doc.created_at)

        // 加权融合
        doc.final_score ← (
            vector_score * W_VECTOR +
            bm25_score * W_BM25 +
            keyword_score * W_KEYWORD +
            quality_score * W_QUALITY +
            time_score * W_TIME
        )
    end for

    return SORT_BY_FINAL_SCORE(documents)

function EMBED_TEXT(text):
    // 使用嵌入模型将文本转换为向量
    tokens ← TOKENIZE(text)
    embedding ← EMBEDDING_MODEL.encode(tokens)
    return embedding

function BM25_SCORE(query, document):
    // 经典 BM25 算法
    score ← 0
    query_terms ← TOKENIZE(query)
    doc_terms ← TOKENIZE(document)
    doc_length ← LENGTH(doc_terms)
    avg_doc_length ← GET_AVG_DOC_LENGTH()

    for each term in query_terms do
        tf ← COUNT(term, doc_terms)
        df ← DOCUMENT_FREQUENCY(term)
        idf ← LOG((N - df + 0.5) / (df + 0.5) + 1)

        // BM25 公式
        numerator ← tf * (K1 + 1)
        denominator ← tf + K1 * (1 - B + B * doc_length / avg_doc_length)
        score += idf * numerator / denominator
    end for

    return score
```

### 复杂度分析

| 复杂度类型 | 表达式 | 说明 |
|-----------|--------|------|
| 时间复杂度（FAISS Flat） | O(D × N) | D=维度, N=向量数 |
| 时间复杂度（FAISS IVF） | O(D × √N + D × K × C) | C=聚类数, K=返回数 |
| 时间复杂度（FAISS HNSW） | O(D × log N × M) | M=图连接数 |
| 时间复杂度（BM25） | O(Q × L) | Q=查询词数, L=文档长度 |
| 时间复杂度（总体） | O(D × log N + K × L) | 使用 HNSW 索引 |
| 空间复杂度 | O(N × D + N × M) | 向量存储 + 图结构 |

不同 FAISS 索引类型的权衡：

| 索引类型 | 搜索速度 | 内存占用 | 召回率 | 适用规模 |
|---------|---------|---------|--------|---------|
| Flat | 慢 | 低 | 100% | <100K |
| IVF | 中 | 中 | >95% | 100K-1M |
| HNSW | 快 | 高 | >99% | <10M |
| PQ | 快 | 极低 | >90% | >1M |

### 优化建议

1. **索引选择**：根据数据量选择合适索引类型（<100K 用 Flat，>100K 用 IVF+HNSW）
2. **量化压缩**：使用 PQ（乘积量化）减少内存占用
3. **批量查询**：多个查询合并为矩阵运算，利用 SIMD 加速
4. **缓存热门查询**：对高频查询结果进行缓存
5. **增量更新**：支持增量添加向量，避免全量重建索引

### 相关代码位置

```
src/local_deep_research/vector_store/faiss_index.py
src/local_deep_research/vector_store/reranker.py
src/local_deep_research/vector_store/embedding.py
```

---

## 14.4 文本分块与嵌入算法

### 问题描述

长文档无法直接作为 LLM 上下文（受 token 限制），需要分割为适当大小的块（chunk），同时保持语义完整性。分块质量直接影响 RAG 检索效果。

### 算法思路

采用递归字符分块（Recursive Character Chunking）策略：

1. **多级分隔符**：按段落→句子→短语→单词的优先级分割
2. **重叠窗口**：相邻块之间保持一定重叠，避免信息断裂
3. **长度约束**：每个块不超过最大 token 限制
4. **语义边界**：优先在语义边界处分割

### 完整伪代码

```
算法: RecursiveCharacterChunking
输入: text (原始文本), max_chunk_size (最大块大小), overlap (重叠大小)
输出: chunks (文本块列表)

function RECURSIVE_CHUNKING(text, max_chunk_size, overlap):
    // 定义分隔符优先级（从高到低）
    separators ← ["\n\n", "\n", ". ", "! ", "? ", "; ", ", ", " ", ""]

    chunks ← CHUNK_BY_SEPARATOR(text, separators, max_chunk_size, overlap)
    return chunks

function CHUNK_BY_SEPARATOR(text, separators, max_chunk_size, overlap):
    if LENGTH(text) ≤ max_chunk_size then
        return [text]
    end if

    if separators = ∅ then
        // 无分隔符可用，强制按字符分割
        return SLICE_TEXT(text, max_chunk_size, overlap)
    end if

    separator ← FIRST(separators)
    remaining_separators ← REST(separators)

    // 按当前分隔符分割
    parts ← SPLIT(text, separator)

    // 合并小块，拆分大块
    chunks ← []
    current_chunk ← ""

    for each part in parts do
        if LENGTH(current_chunk + separator + part) ≤ max_chunk_size then
            // 可以合并到当前块
            if current_chunk ≠ "" then
                current_chunk ← current_chunk + separator + part
            else
                current_chunk ← part
            end if
        else
            // 当前块已满，保存并开始新块
            if current_chunk ≠ "" then
                chunks.APPEND(current_chunk)
                // 添加重叠
                if overlap > 0 AND LENGTH(chunks) > 0 then
                    overlap_text ← GET_OVERLAP_TEXT(current_chunk, overlap)
                    current_chunk ← overlap_text + separator + part
                else
                    current_chunk ← part
                end if
            else
                // 单个部分超过最大大小，递归使用下一个分隔符
                sub_chunks ← CHUNK_BY_SEPARATOR(
                    part, remaining_separators, max_chunk_size, overlap
                )
                chunks.EXTEND(sub_chunks)
                current_chunk ← ""
            end if
        end if
    end for

    if current_chunk ≠ "" then
        chunks.APPEND(current_chunk)
    end if

    return chunks

function GET_OVERLAP_TEXT(chunk, overlap_size):
    // 从块末尾提取重叠文本
    if LENGTH(chunk) ≤ overlap_size then
        return chunk
    end if
    return chunk[-overlap_size:]

function SLICE_TEXT(text, chunk_size, overlap):
    // 强制按字符切片
    chunks ← []
    start ← 0
    while start < LENGTH(text) do
        end ← MIN(start + chunk_size, LENGTH(text))
        chunks.APPEND(text[start:end])
        start ← start + chunk_size - overlap
    end while
    return chunks

// === 嵌入生成 ===

function GENERATE_EMBEDDINGS(chunks, model):
    // 批量生成嵌入向量
    embeddings ← []
    batches ← BATCH(chunks, BATCH_SIZE=32)

    for each batch in batches do
        batch_embeddings ← model.encode(batch)
        embeddings.EXTEND(batch_embeddings)
    end for

    return embeddings
```

### 复杂度分析

| 复杂度类型 | 表达式 | 说明 |
|-----------|--------|------|
| 时间复杂度（分块） | O(L × S) | L=文本长度, S=分隔符层级数 |
| 时间复杂度（嵌入） | O(N × T × D) | N=块数, T=平均token数, D=嵌入维度 |
| 空间复杂度 | O(N × D + L) | 嵌入存储 + 原始文本 |

分块参数对效果的影响：

| 参数 | 较小值 | 较大值 | 推荐值 |
|------|--------|--------|--------|
| chunk_size | 上下文不足 | 包含噪声 | 256-512 tokens |
| overlap | 信息断裂 | 冗余存储 | 10-20% of chunk_size |
| 分隔符层级 | 分割过细 | 分割不足 | 4-6 层 |

### 优化建议

1. **语义分块**：使用句子嵌入检测语义边界，在语义变化处分割
2. **自适应大小**：根据内容密度动态调整块大小
3. **标题传播**：将章节标题添加到每个块中，增强上下文
4. **批量嵌入**：使用 GPU 批量生成嵌入，提升吞吐量
5. **增量更新**：仅对变更的文档重新分块和嵌入

### 相关代码位置

```
src/local_deep_research/vector_store/chunking.py
src/local_deep_research/vector_store/embedding.py
src/local_deep_research/vector_store/indexer.py
```

---

## 14.5 引用解析与格式化

### 问题描述

生成的研究报告需要正确引用信息来源。系统需要解析文本中的引用标记，将其转换为标准引用格式，并确保引用与来源的准确对应。

### 算法思路

CitationFormatter 支持 6 种引用格式（APA、MLA、Chicago、IEEE、GB/T 7714、Nature），核心流程：

1. **引用检测**：识别文本中的引用标记（如 [1]、[2,3]、[1-5]）
2. **引用解析**：展开范围引用，去重排序
3. **来源匹配**：将引用编号映射到实际来源
4. **格式转换**：按目标格式生成引用文本
5. **源标签推导**：通过 URLClassifier 自动判断来源类型

### 完整伪代码

```
算法: CitationFormatting
输入: text (含引用标记的文本), sources (来源列表), format_type (引用格式)
输出: formatted_text (格式化后的文本), bibliography (参考文献列表)

function FORMAT_CITATIONS(text, sources, format_type):
    // 阶段1: 提取引用标记
    citations ← EXTRACT_CITATIONS(text)
    // 匹配模式: [1], [2,3], [1-5], [1,3-5,7]

    // 阶段2: 解析引用编号
    citation_numbers ← PARSE_CITATION_NUMBERS(citations)
    // 展开 [1-5] → [1,2,3,4,5]，去重排序

    // 阶段3: 验证引用有效性
    valid_citations ← []
    for each num in citation_numbers do
        if num ≥ 1 AND num ≤ LENGTH(sources) then
            valid_citations.APPEND(num)
        else
            LOG_WARNING(f"引用编号 {num} 超出范围")
        end if
    end for

    // 阶段4: 生成参考文献列表
    bibliography ← []
    for each num in UNIQUE_SORTED(valid_citations) do
        source ← sources[num - 1]
        formatted ← FORMAT_SINGLE_CITATION(source, num, format_type)
        bibliography.APPEND(formatted)
    end for

    // 阶段5: 替换文本中的引用标记
    formatted_text ← REPLACE_CITATIONS(text, citation_numbers, format_type)

    return formatted_text, bibliography

function EXTRACT_CITATIONS(text):
    // 正则匹配引用标记
    pattern ← r"\[(\d+(?:[-,]\d+)*)\]"
    matches ← REGEX_FINDALL(pattern, text)
    return matches

function PARSE_CITATION_NUMBERS(citation_str):
    // 解析 "1,3-5,7" → [1,3,4,5,7]
    numbers ← []
    parts ← SPLIT(citation_str, ",")
    for each part in parts do
        if CONTAINS(part, "-") then
            range_parts ← SPLIT(part, "-")
            start ← INT(range_parts[0])
            end ← INT(range_parts[1])
            numbers.EXTEND(RANGE(start, end + 1))
        else
            numbers.APPEND(INT(part))
        end if
    end for
    return SORT(UNIQUE(numbers))

function FORMAT_SINGLE_CITATION(source, num, format_type):
    switch format_type:
        case "APA":
            return FORMAT_APA(source)
        case "MLA":
            return FORMAT_MLA(source)
        case "IEEE":
            return f"[{num}] {FORMAT_IEEE(source)}"
        case "GB/T_7714":
            return f"[{num}] {FORMAT_GBT7714(source)}"
        case "Nature":
            return f"{num}. {FORMAT_NATURE(source)}"
        default:
            return FORMAT_DEFAULT(source)

function FORMAT_APA(source):
    // APA 7th 格式
    authors ← FORMAT_AUTHORS_APA(source.authors)
    year ← source.year
    title ← source.title
    url ← source.url

    if source.type == "journal" then
        return f"{authors} ({year}). {title}. {source.journal}, {source.volume}({source.issue}), {source.pages}. {url}"
    else if source.type == "web" then
        return f"{authors} ({year}). {title}. Retrieved from {url}"
    else
        return f"{authors} ({year}). {title}. {url}"
    end if

// === 源标签推导 ===

function DERIVE_SOURCE_TAG(url, content):
    // 通过 URL 和内容特征推导来源类型
    classifier ← URLClassifier()
    source_type ← classifier.classify(url)

    switch source_type:
        case "arxiv":
            return "academic"
        case "github":
            return "code"
        case "news":
            return "news"
        case "wikipedia":
            return "encyclopedia"
        case "pdf":
            return "document"
        default:
            // 基于内容特征判断
            if CONTAINS(content, "Abstract") AND CONTAINS(content, "References") then
                return "academic"
            else if CONTAINS(content, "function") AND CONTAINS(content, "class") then
                return "code"
            else
                return "web"
            end if
    end switch
```

### 复杂度分析

| 复杂度类型 | 表达式 | 说明 |
|-----------|--------|------|
| 时间复杂度（提取） | O(L) | L=文本长度，正则扫描 |
| 时间复杂度（解析） | O(C × R) | C=引用数, R=平均范围大小 |
| 时间复杂度（格式化） | O(N × F) | N=来源数, F=格式模板复杂度 |
| 时间复杂度（总体） | O(L + C × R + N × F) | 线性为主 |
| 空间复杂度 | O(N + C) | 来源列表 + 引用映射 |

### 优化建议

1. **增量格式化**：仅重新格式化变更部分的引用
2. **缓存模板**：预编译引用格式模板，避免重复解析
3. **并行处理**：多个来源的格式化可并行执行
4. **智能编号**：根据引用出现顺序自动重新编号
5. **DOI 解析**：自动从 DOI 获取标准引用元数据

### 相关代码位置

```
src/local_deep_research/citation/formatter.py
src/local_deep_research/citation/parser.py
src/local_deep_research/content_fetcher/url_classifier.py
```

---

## 14.6 限流算法

### 问题描述

多 LLM 提供商和搜索引擎 API 都有速率限制（Rate Limit）。系统需要智能地管理请求速率，避免触发限流，同时最大化吞吐量。

### 算法思路

AdaptiveRateLimitTracker 采用自适应 Token Bucket 变体：

1. **令牌桶基础**：每个 API 端点维护一个令牌桶
2. **自适应速率**：根据 429 响应动态调整填充速率
3. **指数退避**：遇到限流时指数增加等待时间
4. **抖动**：添加随机抖动避免雷鸣群效应
5. **优先级队列**：高优先级请求优先获取令牌

### 完整伪代码

```
算法: AdaptiveRateLimit
输入: request (API请求), endpoint (目标端点)
输出: response (API响应)

// 全局状态
buckets: Dict[str, TokenBucket]  // 每个端点一个桶

function ADAPTIVE_LIMIT_REQUEST(request, endpoint):
    bucket ← GET_OR_CREATE_BUCKET(endpoint)

    // 等待直到获得令牌
    while NOT bucket.try_consume(1) do
        wait_time ← bucket.get_wait_time()
        sleep(wait_time)
    end while

    // 执行请求
    response ← EXECUTE(request)

    // 根据响应调整桶参数
    if response.status == 429 then
        // 被限流，降低速率
        bucket.backoff()
        // 重试
        return ADAPTIVE_LIMIT_REQUEST(request, endpoint)
    else if response.status == 200 then
        // 成功，尝试提高速率
        bucket.optimize()
    end if

    // 更新速率限制信息（从响应头）
    if "X-RateLimit-Remaining" in response.headers then
        bucket.update_remaining(
            INT(response.headers["X-RateLimit-Remaining"])
        )
    end if

    return response

class TokenBucket:
    function __init__(self, rate, capacity):
        self.rate ← rate              // 令牌/秒（填充速率）
        self.capacity ← capacity      // 桶容量（突发容量）
        self.tokens ← capacity        // 当前令牌数
        self.last_refill ← NOW()      // 上次填充时间
        self.min_rate ← rate * 0.1    // 最小速率
        self.max_rate ← rate * 2.0    // 最大速率
        self.backoff_count ← 0        // 连续退避次数

    function try_consume(count):
        self.refill()
        if self.tokens ≥ count then
            self.tokens ← self.tokens - count
            return true
        else
            return false
        end if

    function refill():
        now ← NOW()
        elapsed ← now - self.last_refill
        new_tokens ← elapsed * self.rate
        self.tokens ← MIN(self.capacity, self.tokens + new_tokens)
        self.last_refill ← now

    function backoff():
        // 指数退避 + 抖动
        self.backoff_count ← self.backoff_count + 1
        self.rate ← MAX(self.min_rate, self.rate / 2^self.backoff_count)

        // 添加随机抖动 (±25%)
        jitter ← RANDOM(0.75, 1.25)
        self.rate ← self.rate * jitter

    function optimize():
        // 缓慢提高速率（线性增长）
        if self.backoff_count > 0 then
            self.backoff_count ← self.backoff_count - 1
        end if
        self.rate ← MIN(self.max_rate, self.rate * 1.05)

    function get_wait_time():
        deficit ← 1 - self.tokens
        if deficit ≤ 0 then
            return 0
        end if
        return deficit / self.rate

    function update_remaining(remaining):
        // 根据服务端报告的剩余配额调整
        if remaining < 5 then
            // 接近限流，主动降速
            self.rate ← MIN(self.rate, remaining / 10.0)
        end if
```

### 复杂度分析

| 复杂度类型 | 表达式 | 说明 |
|-----------|--------|------|
| 时间复杂度（单次请求） | O(1) | 令牌检查为常数时间 |
| 时间复杂度（含等待） | O(W) | W=等待时间（取决于速率） |
| 空间复杂度 | O(E) | E=端点数量 |
| 自适应收敛 | O(log R) | R=速率调整范围，指数收敛 |

### 优化建议

1. **预热机制**：系统启动时以较低速率开始，逐步提升
2. **多桶协同**：共享配额的端点使用联合桶
3. **预测调整**：基于历史模式预测高峰时段
4. **优先级借用**：低优先级桶的空闲令牌可借给高优先级
5. **分布式扩展**：多实例场景使用 Redis 共享桶状态

### 相关代码位置

```
src/local_deep_research/llm/rate_limiter.py
src/local_deep_research/search_engines/rate_limiter.py
src/local_deep_research/utils/adaptive_limit.py
```

---

## 14.7 新闻推荐评分算法

### 问题描述

新闻推荐功能需要根据用户的历史阅读行为和显式偏好，对新闻文章进行个性化排序，确保推荐内容既相关又多样。

### 算法思路

采用多因子评分模型：

1. **用户偏好建模**：基于阅读历史和显式设置构建用户画像
2. **内容特征提取**：从文章中提取主题、实体、情感等特征
3. **多因子评分**：综合相关性、时效性、多样性、可信度等因子
4. **探索-利用平衡**：在推荐已知偏好内容和探索新主题间平衡

### 完整伪代码

```
算法: NewsRecommendationScoring
输入: articles (候选文章列表), user_profile (用户画像), config (配置)
输出: ranked_articles (排序后的推荐列表)

function SCORE_ARTICLES(articles, user_profile, config):
    scored ← []

    for each article in articles do
        score ← CALCULATE_ARTICLE_SCORE(article, user_profile, config)
        article.score ← score
        scored.APPEND(article)
    end for

    // 多样性重排序
    ranked ← DIVERSITY_RERANK(scored, config.top_k, config.diversity_weight)
    return ranked

function CALCULATE_ARTICLE_SCORE(article, user_profile, config):
    // 因子1: 主题相关性
    topic_score ← TOPIC_RELEVANCE(article.topics, user_profile.interests)

    // 因子2: 时效性
    recency_score ← RECENCY_DECAY(article.published_at, config.half_life)

    // 因子3: 来源可信度
    credibility_score ← SOURCE_CREDIBILITY(article.source)

    // 因子4: 阅读历史匹配
    history_score ← HISTORY_MATCH(article, user_profile.reading_history)

    // 因子5: 社交热度
    popularity_score ← NORMALIZE(article.engagement_score)

    // 因子6: 探索奖励（新主题奖励）
    exploration_score ← EXPLORATION_BONUS(article.topics, user_profile)

    // 加权融合
    final_score ← (
        topic_score * config.w_topic +
        recency_score * config.w_recency +
        credibility_score * config.w_credibility +
        history_score * config.w_history +
        popularity_score * config.w_popularity +
        exploration_score * config.w_exploration
    )

    return final_score

function TOPIC_RELEVANCE(article_topics, user_interests):
    // 计算文章主题与用户兴趣的余弦相似度
    if article_topics = ∅ OR user_interests = ∅ then
        return 0.5  // 默认中等相关
    end if

    // 使用主题嵌入计算相似度
    article_vec ← AVERAGE_EMBEDDINGS(article_topics)
    user_vec ← AVERAGE_EMBEDDINGS(user_interests)
    return COSINE_SIMILARITY(article_vec, user_vec)

function RECENCY_DECAY(published_at, half_life):
    // 指数衰减
    age_hours ← (NOW() - published_at) / 3600
    return EXP(-LN(2) * age_hours / half_life)

function HISTORY_MATCH(article, reading_history):
    // 基于用户历史阅读的协同过滤
    if reading_history = ∅ then
        return 0.5
    end if

    // 找到与当前文章相似的历史文章
    similar_articles ← FIND_SIMILAR(article, reading_history, k=10)
    if similar_articles = ∅ then
        return 0.5
    end if

    // 基于相似文章的用户评分计算预测评分
    total_score ← 0
    total_weight ← 0
    for each sim in similar_articles do
        weight ← sim.similarity
        total_score += weight * sim.user_rating
        total_weight += weight
    end for

    return total_score / total_weight

function EXPLORATION_BONUS(article_topics, user_profile):
    // 对用户较少接触的主题给予探索奖励
    novelty ← 0
    for each topic in article_topics do
        exposure ← user_profile.topic_exposure.get(topic, 0)
        novelty += 1.0 / (1.0 + exposure)
    end for
    novelty ← novelty / LENGTH(article_topics)

    // 探索率随时间衰减
    exploration_rate ← user_profile.base_exploration_rate
    return novelty * exploration_rate

function DIVERSITY_RERANK(articles, top_k, diversity_weight):
    // 基于类别的多样性重排序
    selected ← []
    category_count ← {}

    for each article in SORT_BY_SCORE(articles) do
        category ← article.category
        category_penalty ← category_count.get(category, 0) * diversity_weight
        adjusted_score ← article.score - category_penalty

        if LENGTH(selected) < top_k then
            selected.APPEND(article)
            category_count[category] ← category_count.get(category, 0) + 1
        end if
    end for

    return selected
```

### 复杂度分析

| 复杂度类型 | 表达式 | 说明 |
|-----------|--------|------|
| 时间复杂度（评分） | O(A × (T + H × S)) | A=文章数, T=主题数, H=历史数, S=相似度计算 |
| 时间复杂度（排序） | O(A log A) | 按分数排序 |
| 时间复杂度（重排序） | O(A × C) | C=类别数 |
| 空间复杂度 | O(A + T + H) | 文章、主题、历史存储 |

### 优化建议

1. **预计算**：用户画像和文章特征离线预计算
2. **近似最近邻**：使用 FAISS 加速相似文章查找
3. **在线学习**：根据用户实时反馈调整权重
4. **冷启动**：新用户基于热门和编辑精选推荐
5. **A/B 测试**：持续优化权重参数

### 相关代码位置

```
src/local_deep_research/news/recommender.py
src/local_deep_research/news/scoring.py
src/local_deep_research/news/user_profile.py
```

---

## 14.8 出口策略评估算法

### 问题描述

出口策略（Egress Policy）用于控制应用可以访问的网络范围，防止 SSRF 攻击和敏感数据泄露。需要根据配置的策略规则，判断特定网络请求是否被允许。

### 算法思路

采用 PDP（Policy Decision Point）决策流程：

1. **DNS 解析**：将目标主机名解析为 IP 地址
2. **IP 分类**：判断 IP 所属范围（公网、私组、回环、链路本地等）
3. **策略匹配**：按优先级匹配允许/拒绝规则
4. **默认拒绝**：未匹配任何规则时默认拒绝

### 完整伪代码

```
算法: EgressPolicyEvaluation
输入: target_url (目标URL), policy (出口策略配置)
输出: decision (允许/拒绝), reason (决策理由)

function EVALUATE_EGRESS(target_url, policy):
    // 阶段1: 解析目标
    hostname ← EXTRACT_HOSTNAME(target_url)
    port ← EXTRACT_PORT(target_url)

    // 阶段2: DNS 解析
    try:
        ip_addresses ← DNS_RESOLVE(hostname)
    except DNS_ERROR:
        return DENY, "DNS解析失败"

    // 阶段3: 对每个解析出的 IP 进行策略评估
    for each ip in ip_addresses do
        decision ← EVALUATE_IP(ip, policy)
        if decision.action == DENY then
            return decision  // 任一 IP 被拒绝则整体拒绝
        end if
    end for

    // 所有 IP 都通过
    return ALLOW, "所有IP通过策略检查"

function EVALUATE_IP(ip, policy):
    // 步骤1: 检查 IP 类型
    ip_class ← CLASSIFY_IP(ip)

    // 步骤2: 按优先级匹配规则（deny 优先于 allow）
    // 先检查拒绝规则
    for each rule in policy.deny_rules do
        if MATCHES(ip, rule) then
            return DENY, f"匹配拒绝规则: {rule.description}"
        end if
    end for

    // 再检查允许规则
    for each rule in policy.allow_rules do
        if MATCHES(ip, rule) then
            return ALLOW, f"匹配允许规则: {rule.description}"
        end if
    end for

    // 步骤3: 默认策略
    if policy.default_action == "deny" then
        return DENY, "未匹配任何规则，默认拒绝"
    else
        return ALLOW, "未匹配任何规则，默认允许"
    end if

function CLASSIFY_IP(ip):
    // 分类 IP 地址类型
    if IS_LOOPBACK(ip) then       // 127.0.0.0/8, ::1
        return "loopback"
    else if IS_PRIVATE(ip) then   // 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16
        return "private"
    else if IS_LINK_LOCAL(ip) then // 169.254.0.0/16, fe80::/10
        return "link_local"
    else if IS_MULTICAST(ip) then  // 224.0.0.0/4
        return "multicast"
    else if IS_RESERVED(ip) then   // 其他保留地址
        return "reserved"
    else
        return "public"
    end if

function MATCHES(ip, rule):
    switch rule.type:
        case "cidr":
            return IP_IN_CIDR(ip, rule.cidr)
        case "range":
            return IP_IN_RANGE(ip, rule.start, rule.end)
        case "classification":
            return CLASSIFY_IP(ip) == rule.classification
        case "hostname":
            return HOSTNAME_MATCH(ip, rule.pattern)
        default:
            return false
    end switch

// === 默认安全策略 ===

function GET_DEFAULT_POLICY():
    policy ← EgressPolicy()

    // 默认拒绝所有私有地址（防 SSRF）
    policy.deny_rules.APPEND(Rule(
        type="classification",
        classification="private",
        description="拒绝私有网络地址"
    ))

    policy.deny_rules.APPEND(Rule(
        type="classification",
        classification="loopback",
        description="拒绝回环地址"
    ))

    policy.deny_rules.APPEND(Rule(
        type="classification",
        classification="link_local",
        description="拒绝链路本地地址"
    ))

    // 允许公共网络
    policy.allow_rules.APPEND(Rule(
        type="classification",
        classification="public",
        description="允许公共网络"
    ))

    policy.default_action ← "deny"
    return policy
```

### 复杂度分析

| 复杂度类型 | 表达式 | 说明 |
|-----------|--------|------|
| 时间复杂度（DNS 解析） | O(D) | D=DNS 解析延迟（网络 I/O） |
| 时间复杂度（IP 分类） | O(1) | 位运算判断 |
| 时间复杂度（规则匹配） | O(R) | R=规则数量 |
| 时间复杂度（总体） | O(D + R) | 线性于规则数 |
| 空间复杂度 | O(R) | 规则存储 |

### 优化建议

1. **DNS 缓存**：缓存 DNS 解析结果，减少重复查询
2. **规则索引**：按 IP 范围建立索引，加速匹配
3. **预编译规则**：将 CIDR 规则编译为高效的匹配结构
4. **异步 DNS**：使用异步 DNS 解析避免阻塞
5. **审计日志**：记录所有出口请求的决策过程

### 相关代码位置

```
src/local_deep_research/security/egress_policy.py
src/local_deep_research/security/ssrf_protection.py
src/local_deep_research/security/dns_resolver.py
```

---

## 算法复杂度总览

| 算法 | 时间复杂度 | 空间复杂度 | 瓶颈 |
|------|-----------|-----------|------|
| 迭代深化搜索 | O(I×Q×E×R) | O(N×M) | LLM 调用延迟 |
| 候选排序 | O(N²×D + N×L) | O(N×D) | LLM 评分 |
| 向量检索 | O(D×log N) | O(N×D) | 嵌入维度 |
| 文本分块 | O(L×S) | O(N×D) | 嵌入生成 |
| 引用格式化 | O(L + C×R) | O(N+C) | 正则匹配 |
| 自适应限流 | O(1) | O(E) | 等待时间 |
| 新闻推荐 | O(A×(T+H×S)) | O(A+T+H) | 相似度计算 |
| 出口策略 | O(D + R) | O(R) | DNS 解析 |

---

☕️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../res/promotion.jpg)