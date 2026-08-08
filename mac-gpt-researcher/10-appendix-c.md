# 附录C 关键算法伪代码与复杂度分析

> **文件**: `docs/wangbin/10-appendix-c.md`  
> **预计 Token**: ~10,000  
> **核心内容**: 深度研究递归、上下文压缩、MCP 两阶段选择、爬虫调度

---

## C.1 深度研究递归算法

### C.1.1 算法描述

深度研究采用**广度优先递归探索**，逐层生成查询并收集发现。

### C.1.2 伪代码

```
算法: DeepResearch(query, breadth, depth, concurrency)
输入: 
    query: 原始研究查询
    breadth: 每层生成的查询数 (默认 3)
    depth: 递归深度 (默认 2)
    concurrency: 并发数 (默认 4)
输出:
    context: 累积的研究发现列表

1.  function DEEP_RESEARCH(query, breadth, depth):
2.      all_learnings ← []
3.      all_urls ← {}
4.      
5.      for current_depth from 1 to depth:
6.          if current_depth == 1:
7.              queries ← GENERATE_QUERIES(query, breadth)
8.          else:
9.              queries ← []
10.             for each learning in all_learnings:
11.                 follow_ups ← GENERATE_FOLLOWUPS(learning, breadth)
12.                 queries.extend(follow_1.             end for
13.            end if
14.            
15.            # 并行执行查询（限制并发数）
16.            results ← PARALLEL_EXECUTE(queries, concurrency):
17.                for each q in queries:
18.                    urls ← SEARCH(q)
18.                    for each url in urls:
19.                        if url not in all_urls:
20.                            content ← SCRAPE(url)
21.                            all_urls.add(url)
22.                            learning ← ANALYZE(content)
23.                            all_learnings.append(learning)
24.                        end if
25.                    end for
26.                end for
27.            end PARALLEL_EXECUTE
28.            
29.            # 报告进度
30.            REPORT_PROGRESS(current_depth, depth, len(queries))
31.        end for
32.        
33.        return all_learnings
34.    end function

35.    function GENERATE_QUERIES(query, n):
36.        prompt ← "Generate {n} research queries for: {query}"
37.        response ← LLM_CALL(prompt, model=strategic)
38.        queries ← PARSE_JSON(response)  # [{query, researchGoal}]
39.        return queries[:n]
40.    end function

41.    function GENERATE_FOLLOWUPS(learning, n):
42.        prompt ← "Based on this learning, generate {n} follow-up questions"
43.        response ← LLM_CALL(prompt, model=strategic)
44.        questions ← PARSE_JSON(response)
45.        return questions[:n]
46.    end function

47.    function PARALLEL_EXECUTE(tasks, concurrency):
48.        semaphore ← Semaphore(concurrency)
49.        results ← []
50.        
51.        async def execute_with_limit(task):
52.            async with semaphore:
53.                return await task
54.            end async with
55.        end async def
56.        
57.        results ← asyncio.gather(*[
58.            execute_with_limit(t) for t in tasks
59.        ])
60.        return results
61.    end function
```

### C.1.3 复杂度分析

| 指标 | 复杂度 | 说明 |
|------|--------|------|
| **时间复杂度** | O(b^d × (S + C + A)) | b=breadth, d=depth, S=搜索, C=抓取, A=分析 |
| **空间复杂度** | O(b^d × L) | L=平均 learning 长度 |
| **LLM 调用次数** | O(b^d) | 每层每个查询一次 |
| **网络请求数** | O(b^d × R) | R=每查询结果数 |

**默认配置** (breadth=3, depth=2):
- 总查询数: 3 + 9 = 12
- LLM 调用: ~12 次生成 + ~12 次分析 = 24 次
- 网络请求: ~12 × 5 = 60 次（每查询 5 结果）

### C.1.4 优化建议

1. **缓存搜索结果**: 相同查询复用结果
2. **增量抓取**: 仅抓取新 URL
3. **提前终止**: 发现足够信息时停止
4. **自适应深度**: 根据发现质量调整深度

---

## C.2 上下文压缩算法

### C.2.1 算法描述

使用嵌入相似度从大量文档中提取与查询最相关的块。

### C.2.2 伪代码

```
算法: ContextCompression(documents, query, embeddings, max_results, threshold)
输入:
    documents: 文档列表
    query: 搜索查询
    embeddings: 嵌入模型
    max_results: 最大返回结果数 (默认 5)
    threshold: 相似度阈值 (默认 None)
输出:
    relevant_context: 相关上下文文本

1.  function COMPRESS_CONTEXT(documents, query, embeddings, max_results):
2.      # 步骤 1: 文档分块
3.      chunks ← []
4.      for each doc in documents:
5.          doc_chunks ← SPLIT_DOCUMENT(doc, chunk_size=1000, overlap=200)
6.          chunks.extend(doc_chunks)
7.      end for
8.      
9.      # 步骤 2: 计算查询嵌入
10.     query_embedding ← embeddings.embed(query)
11.     
12.     # 步骤 3: 计算所有块的嵌入
13.     chunk_embeddings ← []
14.     for each chunk in chunks:
15.         chunk_emb ← embeddings.embed(chunk.content)
16.         chunk_embeddings.append(chunk_emb)
17.     end for
18.     
19.     # 步骤 4: 计算余弦相似度
20.     similarities ← []
21.     for i, chunk_emb in enumerate(chunk_embeddings):
22.         sim ← COSINE_SIMILARITY(query_embedding, chunk_emb)
23.         similarities.append((i, sim))
24.     end for
25.     
26.     # 步骤 5: 排序并过滤
27.     similarities.sort(key=lambda x: x[1], reverse=True)
28.     filtered ← [s for s in similarities if s[1] >= threshold]
29.     
30.     # 步骤 6: 返回最相关的块
31.     result ← []
32.     for i, sim in filtered[:max_results]:
33.         result.append(chunks[i].content)
34.     end for
35.     
36.     return "\n\n".join(result)
37. end function

38. function SPLIT_DOCUMENT(doc, chunk_size, overlap):
39.     text ← doc.content
40.     chunks ← []
41.     start ← 0
42.     
43.     while start < len(text):
44.         end ← min(start + chunk_size, len(text))
45.         chunks.append(text[start:end])
46.         start ← start + (chunk_size - overlap)
47.     end while
48.     
49.     return chunks
50. end function

51. function COSINE_SIMILARITY(a, b):
52.     dot_product ← sum(a[i] * b[i] for i in range(len(a)))
53.     norm_a ← sqrt(sum(a[i]^2 for i in range(len(a))))
54.     norm_b ← sqrt(sum(b[i]^2 for i in range(len(b))))
55.     return dot_product / (norm_a * norm_b)
56. end function
```

### C.2.3 复杂度分析

| 指标 | 复杂度 | 说明 |
|------|--------|------|
| **分块** | O(D) | D=文档总字符数 |
| **嵌入计算** | O(C × E) | C=块数, E=嵌入维度 |
| **相似度计算** | O(C × E) | 每块与查询计算 |
| **排序** | O(C log C) | 按相似度排序 |
| **总时间** | O(C × E + C log C) | 主导项是嵌入计算 |
| **空间** | O(C × E) | 存储所有嵌入 |

### C.2.4 优化建议

1. **批量嵌入**: 一次 API 调用嵌入多个块
2. **近似最近邻**: 使用 FAISS/Annoy 加速搜索
3. **嵌入缓存**: 缓存已计算的嵌入
4. **分层过滤**: 先关键词过滤再嵌入

---

## C.3 MCP 两阶段工具选择算法

### C.3.1 算法描述

从大量 MCP 工具中选择最相关的工具，然后使用这些工具执行研究。

### C.3.2 伪代码

```
算法: MCP_TwoStage_Search(query, mcp_configs, max_tools)
输入:
    query: 研究查询
    mcp_configs: MCP 服务器配置列表
    max_tools: 最大选择工具数 (默认 3)
输出:
    research_results: 研究结果

1.  function MCP_SEARCH(query, mcp_configs, max_tools):
2.      # 阶段 1: 获取所有可用工具
3.      client ← MCPClientManager(mcp_configs)
4.      all_tools ← client.get_all_tools()
5.      
6.      # 阶段 2: LLM 工具选择
7.      selected ← SELECT_TOOLS(query, all_tools, max_tools)
8.      
9.      # 阶段 3: 绑定工具并执行研究
10.     llm_with_tools ← BIND_TOOLS(llm, selected)
11.     results ← EXECUTE_RESEARCH(llm_with_tools, query)
12.     
13.     return results
14. end function

15. function SELECT_TOOLS(query, all_tools, max_tools):
16.     # 构建工具信息
17.     tools_info ← []
18.     for i, tool in enumerate(all_tools):
19.         tools_info.append({
20.             "index": i,
21.             "name": tool.name,
22.             "description": tool.description
23.         })
24.     end for
25.     
26.     # LLM 选择
27.     prompt ← f"""
28.     Query: {query}
29.     Available tools: {tools_info}
33.     Select {max_tools} most relevant tools.
34.     Return JSON: {{"selected_tools": [...]}}
35.     """
36.     
37.     response ← LLM_CALL(prompt, temperature=0.2)
38.     selected ← PARSE_JSON(response)
39.     
40.     return selected.selected_tools
41. end function

42. function EXECUTE_RESEARCH(llm_with_tools, query):
43.     messages ← [HumanMessage(content=query)]
44.     
45.     # 第一轮 LLM 调用（可能触发工具调用）
46.     response ← llm_with_tools.ainvoke(messages)
47.     
48.     # 处理工具调用
49.     while response has tool_calls:
50.         messages.append(response)
51.         
52.         for each tool_call in response.tool_calls:
53.             tool ← FIND_TOOL(tool_call.name)
54.             result ← EXECUTE_TOOL(tool, tool_call.args)
55.             messages.append(ToolMessage(result, tool_call.id))
56.         end for
57.         
58.         # 获取最终响应
59.         response ← llm_with_tools.ainvoke(messages)
60.     end while
61.     
62.     return response.content
63. end function
```

### C.3.3 复杂度分析

| 指标 | 复杂度 | 说明 |
|------|--------|------|
| **工具获取** | O(S) | S=MCP 服务器数 |
| **工具选择** | O(T) | T=总工具数（LLM 处理） |
| **研究执行** | O(K × (L + X)) | K=工具调用次数, L=LLM, X=工具执行 |
| **总时间** | O(T + K × (L + X)) | 主导是工具执行 |

### C.3.4 优化建议

1. **工具缓存**: 缓存工具列表
2. **并行工具调用**: 多个工具同时执行
3. **工具预过滤**: 关键词匹配粗筛

---

## C.4 爬虫调度与限速算法

### C.4.1 算法描述

使用 WorkerPool 控制并发，GlobalRateLimiter 控制请求频率。

### C.4.2 伪代码

```
算法: ScrapeURLs(urls, max_workers, rate_limit_delay)
输入:
    urls: URL 列表
    max_workers: 最大并发数 (默认 15)
    rate_limit_delay: 请求间隔 (默认 0)
输出:
    results: 抓取结果列表

1.  function SCRAPE_URLS(urls, max_workers, rate_limit_delay):
2.      # URL 去重
3.      unique_urls ← REMOVE_DUPLICATES(urls)
4.      
5.      # 创建工作池
6.      pool ← WorkerPool(max_workers, rate_limit_delay)
7.      
8.      # 并行抓取
9.      results ← asyncio.gather(*[
10.         SCRAPE_WITH_THROTTLE(pool, url)
11.         for url in unique_urls
12.     ])
13.     
14.     # 过滤失败结果
15.     successful ← [r for r in results if r.content is not None]
16.     
17.     return successful
18. end function

19. function SCRAPE_WITH_THROTTLE(pool, url):
20.     async with pool.throttle():          # 获取信号量
21.         await WAIT_IF_NEEDED()           # 全局限速
22.         
23.         try:
24.             scraper ← SELECT_SCRAPER(url)  # 选择爬虫
25.             content, images, title ← scraper.scrape()
26.             
27.             if len(content) >= 100:
28.                 return {url, content, images, title}
29.             else:
30.                 return {url, content: None}
31.         except Exception as e:
32.             LOG_ERROR(url, e)
33.             return {url, content: None}
34.         end try
35.     end async with
36. end function

37. function WAIT_IF_NEEDED():
38.     if rate_limit_delay <= 0:
39.         return
40.     end if
41.     
42.     lock ← GET_GLOBAL_LOCK()
43.     async with lock:
44.         now ← current_time()
45.         elapsed ← now - last_request_time
46.         
47.         if elapsed < rate_limit_delay:
48.             sleep(rate_limit_delay - elapsed)
49.         end if
50.         
51.         last_request_time ← current_time()
52.     end async with
53. end function

54. class WorkerPool:
55.     semaphore ← Semaphore(max_workers)
56.     
57.     async def throttle():
58.         async with semaphore:
59.             await WAIT_IF_NEEDED()
60.             yield
61.         end async with
62.     end async def
63. end class
```

### C.4.3 复杂度分析

| 指标 | 复杂度 | 说明 |
|------|--------|------|
| **并发控制** | O(1) | Semaphore 操作 |
| **限速等待** | O(1) | 时间计算和 sleep |
| **总抓取时间** | O(N / W × max(T, D)) | N=URL数, W=Worker数, T=抓取时间, D=限速间隔 |
| **空间** | O(W) | 并发 Worker 数 |

### C.4.4 优化建议

1. **连接池复用**: 复用 HTTP 连接
2. **智能调度**: 优先抓取高价值 URL
3. **自适应限速**: 根据响应调整延迟

---

## C.5 成本计算算法

### C.5.1 算法描述

根据 token 用量和模型定价计算 API 调用成本。

### C.5.2 伪代码

```
算法: CalculateCost(provider, model, input_text, output_text, usage_metadata)
输入:
    provider: LLM 提供商
    model: 模型名称
    input_text: 输入文本
    output_text: 输出文本
    usage_metadata: API 返回的用量元数据
输出:
    cost: 成本 (USD)

1.  function CALCULATE_COST(provider, model, input_text, output_text, usage_metadata):
2.      # 优先级 1: Anthropic 特殊处理
3.      if provider == "anthropic":
4.          anthropic_cost ← CALCULATE_ANTHROPIC_COST(model, usage_metadata)
5.          if anthropic_cost is not None:
6.              return anthropic_cost
7.          end if
8.      end if
9.      
10.     # 优先级 2: 使用 API 报告的 token 用量
11.     if usage_metadata has input_tokens and output_tokens:
12.         input_tokens ← usage_metadata.input_tokens
13.         output_tokens ← usage_metadata.output_tokens
14.     else:
15.         # 优先级 3: tiktoken 估算
16.         encoding ← tiktoken.get_encoding("o200k_base")
17.         input_tokens ← len(encoding.encode(input_text))
18.         output_tokens ← len(encoding.encode(output_text))
19.     end if
20.     
21.     # 查找模型定价
22.     pricing ← FIND_PRICING(model)
23.     # pricing = (input_price_per_mtok, output_price_per_mtok)
24.     
25.     # 计算成本
26.     input_cost ← input_tokens * pricing.input_price / 1_000_000
27.     output_cost ← output_tokens * pricing.output_price / 1_000_000
28.     
29.     return input_cost + output_cost
30. end function

31. function FIND_PRICING(model):
32.     normalized ← model.lower()
33.     
34.     for (patterns, input_price, output_price) in PRICING_TABLE:
35.         if any(pattern in normalized for pattern in patterns):
36.             return (input_price, output_price)
37.         end if
38.     end for
39.     
40.     # 默认定价
41.     return (0.000005, 0.000015)
42. end function
```

### C.5.3 复杂度分析

| 指标 | 复杂度 | 说明 |
|------|--------|------|
| **定价查找** | O(P) | P=定价表条目数 |
| **tiktoken 编码** | O(L) | L=文本长度 |
| **总时间** | O(P + L) | 通常很快 |

---

## C.6 总结

| 算法 | 时间复杂度 | 空间复杂度 | 瓶颈 |
|------|-----------|-----------|------|
| 深度研究 | O(b^d × (S+C+A)) | O(b^d × L) | LLM 调用 |
| 上下文压缩 | O(C×E + C log C) | O(C×E) | 嵌入计算 |
| MCP 选择 | O(T + K×(L+X)) | O(T) | 工具执行 |
| 爬虫调度 | O(N/W × max(T,D)) | O(W) | 网络 I/O |
| 成本计算 | O(P + L) | O(1) | 可忽略 |

---

> **下一节**: → `10-appendix-d.md` — 附录D 测试策略与主要用例

---

☕️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../res/promotion.jpg)