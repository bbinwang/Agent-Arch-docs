# 3. 系统流程与时序图 (System Flows & Sequence Diagrams)

> **章节编号**: 03/13  
> **预计篇幅**: ~15,000 字  
> **核心图表**: 10+ 张 Mermaid 流程图 + 时序图  
> **覆盖范围**: 所有核心业务流程，细化到函数级别

---

## 3.1 流程总览

本系统的业务流程可以分为 **三大阶段**：

| 阶段 | 名称 | 涉及节点 | 输入 | 输出 |
|------|------|----------|------|------|
| **Phase 1** | 计划流水线 (Planning Pipeline) | anonymize → planner → de_anonymize → break_down | 原始问题 | 细化任务列表 |
| **Phase 2** | 执行循环 (Execution Loop) | task_handler → retrieve/answer → replan → can_be_answered | 任务列表 | 聚合上下文 |
| **Phase 3** | 答案生成 (Answer Generation) | get_final_answer | 聚合上下文 | 最终答案 |

```mermaid
flowchart LR
    P1["Phase 1<br/>计划流水线"] --> P2["Phase 2<br/>执行循环"]
    P2 -->|"循环直到可回答"| P2
    P2 -->|"可回答"| P3["Phase 3<br/>答案生成"]
```

---

## 3.2 流程 1: 端到端主流程 (End-to-End Main Flow)

### 3.2.1 流程图

```mermaid
flowchart TD
    START(["用户输入问题"]) --> A["① anonymize_question<br/>functions_for_pipeline.py:984"]
    
    A -->|"输出: anonymized_question, mapping"| B["② planner<br/>functions_for_pipeline.py:1025"]
    B -->|"输出: plan: List[str]"| C["③ de_anonymize_plan<br/>functions_for_pipeline.py:1008"]
    C -->|"输出: plan (还原)"| D["④ break_down_plan<br/>functions_for_pipeline.py:1042"]
    D -->|"输出: refined_plan"| E["⑤ task_handler<br/>functions_for_pipeline.py:803"]
    
    E -->|"Tool A"| F["⑥a retrieve_chunks<br/>子图执行"]
    E -->|"Tool B"| G["⑥b retrieve_summaries<br/>子图执行"]
    E -->|"Tool C"| H["⑥c retrieve_book_quotes<br/>子图执行"]
    E -->|"Tool D"| I["⑥d answer<br/>子图执行"]
    
    F -->|"追加到 aggregated_context"| J["⑦ replan<br/>functions_for_pipeline.py:1059"]
    G --> J
    H --> J
    I --> J
    
    J -->|"更新 plan"| K{"⑧ can_be_answered?<br/>functions_for_pipeline.py:1076"}
    K -->|"否"| D
    K -->|"是"| L["⑨ get_final_answer<br/>functions_for_pipeline.py:963"]
    
    L -->|"输出: response"| END(["返回最终答案"])
    
    style A fill:#e1f5fe
    style E fill:#fff3e0
    style K fill:#f3e5f5
    style L fill:#e8f5e9
```

### 3.2.2 详细步骤说明

| 步骤 | 函数 | 文件位置 | 核心逻辑 | 异常处理 |
|------|------|----------|----------|----------|
| ① | `anonymize_queries()` | :984-1005 | 调用 `anonymize_question_chain`，将人名替换为 X/Y/Z | 无显式处理 |
| ② | `plan_step()` | :1025-1039 | 调用 `planner`，生成步骤化计划 | 无显式处理 |
| ③ | `deanonymize_queries()` | :1008-1022 | 调用 `de_anonymize_plan_chain`，还原变量 | 无显式处理 |
| ④ | `break_down_plan_step()` | :1042-1055 | 调用 `break_down_plan_chain`，细化任务 | 无显式处理 |
| ⑤ | `run_task_handler_chain()` | :803-850 | 调用 `task_handler_chain`，选择工具 | ValueError 如果工具无效 |
| ⑥a-d | `run_qualitative_*_workflow()` | :876-961 | 调用子图流式执行 | 无显式处理 |
| ⑦ | `replan_step()` | :1059-1073 | 调用 `replanner`，更新计划 | 无显式处理 |
| ⑧ | `can_be_answered()` | :1076-1101 | 调用 `can_be_answered_already_chain` | 无显式处理 |
| ⑨ | `run_qualtative_answer_workflow_for_final_answer()` | :963-981 | 调用回答子图 | 无显式处理 |

### 3.2.3 时序图

```mermaid
sequenceDiagram
    participant User as 用户
    participant UI as Streamlit UI<br/>(simulate_agent.py)
    participant Graph as LangGraph<br/>(主图)
    participant LLM as OpenAI GPT-4o
    participant VS as FAISS 向量存储

    User->>UI: 输入问题 "what is the class..."
    UI->>Graph: invoke({"question": ...})
    
    Note over Graph: Phase 1: 计划流水线
    
    Graph->>LLM: anonymize_question_chain.invoke()
    LLM-->>Graph: {anonymized_question, mapping}
    
    Graph->>LLM: planner.invoke(anonymized_question)
    LLM-->>Graph: {plan: ["步骤1", "步骤2", ...]}
    
    Graph->>LLM: de_anonymize_plan_chain.invoke()
    LLM-->>Graph: {plan: ["原始实体步骤1", ...]}
    
    Graph->>LLM: break_down_plan_chain.invoke()
    LLM-->>Graph: {plan: [可执行任务1, ...]}
    
    Note over Graph: Phase 2: 执行循环
    
    loop 每个任务
        Graph->>LLM: task_handler_chain.invoke()
        LLM-->>Graph: {tool: "retrieve_chunks", query: ...}
        
        alt Tool A/B/C (检索)
            Graph->>VS: get_relevant_documents(query)
            VS-->>Graph: [Document(page_content=...)]
            Graph->>LLM: keep_only_relevant_content_chain.invoke()
            LLM-->>Graph: {relevant_content: "..."}
            Graph->>LLM: is_distilled_grounded_chain.invoke()
            LLM-->>Graph: {grounded: true/false}
        else Tool D (回答)
            Graph->>LLM: question_answer_cot_chain.invoke()
            LLM-->>Graph: {answer: "..."}
            Graph->>LLM: is_grounded_on_facts_chain.invoke()
            LLM-->>Graph: {grounded_on_facts: true/false}
        end
        
        Graph->>LLM: replanner.invoke()
        LLM-->>Graph: {plan: [剩余任务...]}
        
        Graph->>LLM: can_be_answered_already_chain.invoke()
        LLM-->>Graph: {can_be_answered: true/false}
    end
    
    Note over Graph: Phase 3: 最终答案
    
    Graph->>LLM: question_answer_cot_chain.invoke(全部上下文)
    LLM-->>Graph: {answer: "最终答案..."}
    
    Graph-->>UI: {response: "最终答案..."}
    UI-->>User: 显示答案 + 可视化
```

---

## 3.3 流程 2: 问题匿名化流程 (Question Anonymization)

### 3.3.1 流程图

```mermaid
flowchart TD
    START(["输入: question = 'how did harry beat quirrell?'"]) --> A["anonymize_queries(state)"]
    
    A --> B["构造 input_values = {'question': state['question']}"]
    B --> C["anonymize_question_chain.invoke(input_values)"]
    
    C --> D{"Chain 内部流程"}
    D --> D1["PromptTemplate 渲染<br/>注入 question"]
    D1 --> D2["ChatOpenAI 调用 GPT-4o"]
    D2 --> D3["JsonOutputParser 解析<br/>JSON → dict"]
    
    D3 --> E["提取 anonymized_question<br/>例: 'how did X beat Y?'"]
    E --> F["提取 mapping<br/>例: {'X': 'harry', 'Y': 'quirrell'}"]
    
    F --> G["更新 state['anonymized_question']"]
    G --> H["更新 state['mapping']"]
    H --> END(["输出: 匿名化状态"])
    
    style C fill:#e3f2fd
    style E fill:#e8f5e9
```

### 3.3.2 详细步骤说明

**步骤 1**: 函数入口 `anonymize_queries(state: PlanExecute)`

- **位置**: `functions_for_pipeline.py:984-1005`
- **输入**: `state["question"]` — 原始自然语言问题
- **输出**: 更新 `state["anonymized_question"]` 和 `state["mapping"]`

**步骤 2**: 调用 `anonymize_question_chain`

该 Chain 的构建（:713-744）：

```python
anonymize_question_chain = anonymize_question_prompt | anonymize_question_llm | anonymize_question_parser
```

- **Prompt**: 包含 2 个示例（"who is harry potter?" → "who is X?"）
- **LLM**: ChatOpenAI(temperature=0, model_name="gpt-4o")
- **Parser**: JsonOutputParser(pydantic_object=AnonymizeQuestion)

**步骤 3**: 输出结构

```python
class AnonymizeQuestion(BaseModel):
    anonymized_question: str  # "how did X beat Y?"
    mapping: dict             # {"X": "harry", "Y": "quirrell"}
    explanation: str          # 解释替换理由
```

### 3.3.3 设计 Rationale

> **为什么需要匿名化？**
>
> 1. **消除先验偏见**: GPT-4o 在预训练中"知道"哈利波特的故事。如果不匿名，LLM 可能基于参数知识而非检索数据制定计划
> 2. **通用性**: 匿名化后的计划更具通用性，可应用于不同领域的同名实体
> 3. **可验证性**: 可以明确区分"LLM 知道什么"和"数据说了什么"
>
> **潜在问题**:
> - 过度匿名化可能丢失语义（如 "who is the Chosen One?" → "who is the X?" 丢失了关键限定）
> - 实体识别错误可能导致错误替换

---

## 3.4 流程 3: 计划生成与细化流程 (Plan Generation & Refinement)

### 3.4.1 流程图

```mermaid
flowchart TD
    START(["输入: anonymized_question"]) --> A["plan_step(state)"]
    
    A --> B["planner.invoke({'question': anonymized_question})"]
    B --> C["GPT-4o 生成步骤化计划"]
    C --> D["输出: Plan(steps=['步骤1', '步骤2', ...])"]
    
    D --> E["deanonymize_queries(state)"]
    E --> F["de_anonymize_plan_chain.invoke({'plan', 'mapping'})"]
    F --> G["变量替换: X→harry, Y→quirrell"]
    
    G --> H["break_down_plan_step(state)"]
    H --> I["break_down_plan_chain.invoke(plan)"]
    I --> J["将每个步骤映射到可执行类型"]
    
    J --> K{"步骤类型判断"}
    K -->|"检索类"| K1["标记为 Tool A/B/C<br/>(chunks/summaries/quotes)"]
    K -->|"回答类"| K2["标记为 Tool D<br/>(answer_from_context)"]
    
    K1 --> END(["输出: refined_plan"])
    K2 --> END
    
    style B fill:#e3f2fd
    style F fill:#e3f2fd
    style I fill:#e3f2fd
```

### 3.4.2 详细步骤说明

**计划生成 (planner)**:
- **位置**: `functions_for_pipeline.py:582-600`
- **Prompt 核心**: "come up with a simple step by step plan"
- **输出**: `Plan(steps=["Identify the villain", "Find who helped the villain", ...])`

**去匿名化 (de_anonymize_plan)**:
- **位置**: `functions_for_pipeline.py:747-766`
- **Prompt 核心**: "replace all the variables in the list of tasks with the mapped words"
- **特殊处理**: 转义引号和新行（"the format is json so escape quotes and new lines"）

**计划细化 (break_down_plan)**:
- **位置**: `functions_for_pipeline.py:603-626`
- **Prompt 核心**: 将每个步骤明确为四种工具之一
- **关键约束**: "every step has to be able to be executed by either i/ii/iii/iv"

### 3.4.3 示例流转

```
输入: "how did harry beat quirrell?"

匿名化后: "how did X beat Y?"  mapping: {"X": "harry", "Y": "quirrell"}

planner 输出:
1. Identify who X is
2. Identify who Y is
3. Find the confrontation between X and Y
4. Determine how X defeated Y

去匿名化后:
1. Identify who harry is
2. Identify who quirrell is
3. Find the confrontation between harry and quirrell
4. Determine how harry defeated quirrell

细化后:
1. [Tool A] Search for information about harry
2. [Tool A] Search for information about quirrell
3. [Tool A] Find confrontation details
4. [Tool D] Answer: how did harry defeat quirrell
```

---

## 3.5 流程 4: 任务路由流程 (Task Routing)

### 3.5.1 流程图

```mermaid
flowchart TD
    START(["输入: curr_task, aggregated_context,<br/>last_tool, past_steps, question"]) --> A["run_task_handler_chain(state)"]
    
    A --> B["构造 inputs dict<br/>(5 个输入字段)"]
    B --> C["task_handler_chain.invoke(inputs)"]
    
    C --> D["GPT-4o 分析任务类型"]
    D --> E{"选择工具"}
    
    E -->|"Tool A"| E1["state['tool'] = 'retrieve_chunks'<br/>state['query'] = output.query"]
    E -->|"Tool B"| E2["state['tool'] = 'retrieve_summaries'<br/>state['query'] = output.query"]
    E -->|"Tool C"| E3["state['tool'] = 'retrieve_quotes'<br/>state['query'] = output.query"]
    E -->|"Tool D"| E4["state['tool'] = 'answer'<br/>state['query'] = output.query<br/>state['curr_context'] = output.curr_context"]
    
    E1 --> F["state['plan'].pop(0)<br/>state['past_steps'].append(curr_task)"]
    E2 --> F
    E3 --> F
    E4 --> F
    
    F --> G["retrieve_or_answer(state)"]
    G --> H{"条件边路由"}
    H -->|"chosen_tool_is_retrieve_chunks"| H1["→ retrieve_chunks 节点"]
    H -->|"chosen_tool_is_retrieve_summaries"| H2["→ retrieve_summaries 节点"]
    H -->|"chosen_tool_is_retrieve_quotes"| H3["→ retrieve_book_quotes 节点"]
    H -->|"chosen_tool_is_answer"| H4["→ answer 节点"]
    
    style C fill:#fff3e0
    style D fill:#fff3e0
```

### 3.5.2 详细步骤说明

**关键代码** (`functions_for_pipeline.py:803-850`):

```python
def run_task_handler_chain(state: PlanExecute):
    state["curr_state"] = "task_handler"
    curr_task = state["plan"][0]  # 取第一个任务
    
    inputs = {
        "curr_task": curr_task,
        "aggregated_context": state["aggregated_context"],
        "last_tool": state["tool"],
        "past_steps": state["past_steps"],
        "question": state["question"]
    }
    
    output = task_handler_chain.invoke(inputs)
    
    state["past_steps"].append(curr_task)
    state["plan"].pop(0)  # 移除已处理任务
    
    # 根据工具类型更新状态
    if output.tool == "retrieve_chunks":
        state["query_to_retrieve_or_answer"] = output.query
        state["tool"] = "retrieve_chunks"
    # ... 其他工具类似
```

**路由逻辑** (`functions_for_pipeline.py:854-872`):

```python
def retrieve_or_answer(state: PlanExecute):
    if state["tool"] == "retrieve_chunks":
        return "chosen_tool_is_retrieve_chunks"
    # ... 其他工具
```

> **设计 Rationale**: 将路由逻辑拆分为两步（`run_task_handler_chain` + `retrieve_or_answer`）是因为：
> 1. **关注点分离**: 前者处理 LLM 决策，后者处理图路由
> 2. **可测试性**: 可以独立测试 LLM 决策和路由映射
> 3. **可扩展性**: 新增工具只需添加新的条件分支

---

## 3.6 流程 5: 检索子图流程 (Qualitative Retrieval Sub-graph)

### 3.6.1 流程图

```mermaid
flowchart TD
    START(["输入: question (query)"]) --> A["retrieve_*_context_per_question(state)"]
    
    A --> B["chunks_query_retriever.get_relevant_documents(question)"]
    B --> C["FAISS 向量检索<br/>返回 top-k 相似文档"]
    C --> D["拼接文档内容<br/>context = ' '.join(doc.page_content)"]
    D --> D1["escape_quotes(context)<br/>转义引号"]
    
    D1 --> E["keep_only_relevant_content(state)"]
    E --> F["构造 input_data = {'query', 'retrieved_documents'}"]
    F --> G["keep_only_relevant_content_chain.invoke(input_data)"]
    
    G --> H["GPT-4o 蒸馏<br/>过滤无关句子"]
    H --> I["输出: relevant_content"]
    
    I --> J["is_distilled_content_grounded_on_content(state)"]
    J --> K["构造 input_data = {'distilled_content', 'original_context'}"]
    K --> L["is_distilled_content_grounded_on_content_chain.invoke(input_data)"]
    
    L --> M{"蒸馏内容是否基于原始检索?"}
    M -->|"是 (grounded)"| M1["返回 END<br/>输出: relevant_context"]
    M -->|"否 (not grounded)"| M2["循环回 keep_only_relevant_content<br/>重新蒸馏"]
    
    M2 --> E
    
    style G fill:#e3f2fd
    style L fill:#e3f2fd
    style M fill:#f3e5f5
```

### 3.6.2 三种检索子图的差异

| 维度 | Chunks 子图 | Summaries 子图 | Quotes 子图 |
|------|-------------|----------------|-------------|
| **检索器** | `chunks_query_retriever` | `chapter_summaries_query_retriever` | `book_quotes_query_retriever` |
| **k 值** | 1 | 1 | 10 |
| **元数据** | 无 | chapter 编号 | 无 |
| **蒸馏** | 共享 `keep_only_relevant_content` | 同左 | 同左 |
| **验证** | 共享 `is_distilled_content_grounded_on_content` | 同左 | 同左 |
| **创建函数** | `:437-458` | `:461-482` | `:485-506` |

### 3.6.3 时序图

```mermaid
sequenceDiagram
    participant Graph as 主图
    participant Sub as 检索子图
    participant VS as FAISS
    participant LLM as GPT-4o

    Graph->>Sub: stream({"question": "query"})
    
    Sub->>Sub: retrieve_chunks_context_per_question()
    Sub->>VS: get_relevant_documents("query")
    VS-->>Sub: [Document(page_content="...")]
    Sub->>Sub: context = join(docs)
    
    Sub->>Sub: keep_only_relevant_content()
    Sub->>LLM: keep_only_relevant_content_chain.invoke()
    Note right of LLM: 蒸馏: 过滤无关句子
    LLM-->>Sub: {relevant_content: "..."}
    
    Sub->>Sub: is_distilled_content_grounded_on_content()
    Sub->>LLM: is_distilled_grounded_chain.invoke()
    Note right of LLM: 验证: 蒸馏是否基于原始?
    LLM-->>Sub: {grounded: true/false}
    
    alt grounded = true
        Sub-->>Graph: {relevant_context: "..."}
    else grounded = false
        Note over Sub: 循环回 keep_only_relevant_content
        Sub->>LLM: 重新蒸馏
        LLM-->>Sub: {relevant_content: "..."}
    end
```

---

## 3.7 流程 6: 回答子图流程 (Answer Sub-graph)

### 3.7.1 流程图

```mermaid
flowchart TD
    START(["输入: question, context"]) --> A["answer_question_from_context(state)"]
    
    A --> B["构造 input_data = {'question', 'context'}"]
    B --> C["question_answer_from_context_cot_chain.invoke(input_data)"]
    
    C --> D["GPT-4o 使用 CoT 推理"]
    D --> D1["Prompt 包含 3 个示例:<br/>1. 身高比较<br/>2. 魔法咒语<br/>3. 无法回答示例"]
    D1 --> D2["生成 reasoning chain"]
    D2 --> D3["输出 final answer"]
    
    D3 --> E["is_answer_grounded_on_context(state)"]
    E --> F["构造 input_data = {'context', 'answer'}"]
    F --> G["is_grounded_on_facts_chain.invoke(input_data)"]
    
    G --> H{"答案是否基于上下文?"}
    H -->|"是 (grounded)"| H1["返回 END<br/>输出: answer"]
    H -->|"否 (hallucination)"| H2["循环回 answer_question_from_context<br/>重新生成"]
    
    H2 --> A
    
    style C fill:#e3f2fd
    style G fill:#e3f2fd
    style H fill:#f3e5f5
```

### 3.7.2 CoT Prompt 设计

回答子图的 Prompt 是项目中最精心设计的 Prompt 之一：

```
Examples of Chain-of-Thought Reasoning

Example 1: 身高比较 (简单推理)
Example 2: 魔法咒语 (多步推理)
Example 3: 无法回答示例 (拒绝推理 - 关键!)

Context: {context}
Question: {question}
```

> **设计 Rationale**: Example 3（无法回答示例）是**刻意设计**的负例：
> - 它教会模型：当上下文不足以回答时，应承认不知道
> - 这是防止幻觉的关键机制
> - 示例中明确说："there is no way to determine the reason"

### 3.7.3 幻觉检测机制

```python
def is_answer_grounded_on_context(state):
    context = state["context"]
    answer = state["answer"]
    result = is_grounded_on_facts_chain.invoke({"context": context, "answer": answer})
    grounded_on_facts = result.grounded_on_facts
    
    if not grounded_on_facts:
        return "hallucination"  # 触发循环
    else:
        return "grounded on context"  # 结束子图
```

---

## 3.8 流程 7: 重规划流程 (Re-plan)

### 3.8.1 流程图

```mermaid
flowchart TD
    START(["输入: question, plan, past_steps,<br/>aggregated_context"]) --> A["replan_step(state)"]
    
    A --> B["构造 inputs dict"]
    B --> C["replanner.invoke(inputs)"]
    
    C --> D["GPT-4o 分析"]
    D --> D1["回顾已完成步骤 (past_steps)"]
    D2["审视当前计划 (plan)"]
    D3["评估已聚合信息 (aggregated_context)"]
    D4["决定下一步计划"]
    
    D1 & D2 & D3 & D4 --> E["输出: Plan(steps=[剩余步骤])"]
    
    E --> F["更新 state['plan']"]
    F --> G["can_be_answered(state)"]
    
    G --> H["can_be_answered_already_chain.invoke()"]
    H --> I{"可回答?"}
    I -->|"是"| I1["→ get_final_answer"]
    I -->|"否"| I2["→ break_down_plan (继续循环)"]
    
    style C fill:#e3f2fd
    style H fill:#e3f2fd
    style I fill:#f3e5f5
```

### 3.8.2 Replanner Prompt 设计

```python
replanner_prompt_template ="""
For the given objective, come up with a simple step by step plan...

assume that the answer was not found yet and you need to update the plan accordingly,
so the plan should never be empty.

Your objective was this: {question}
Your original plan was this: {plan}
You have currently done the follow steps: {past_steps}
You already have the following context: {aggregated_context}

Update your plan accordingly. If further steps are needed, fill out the plan with only those steps.
Do not return previously done steps as part of the plan.
"""
```

> **关键约束**: "assume that the answer was not found yet" + "the plan should never be empty"
> - 这强制 replanner 始终产出下一步计划
> - 即使上下文已经足够，也需要通过 `can_be_answered` 显式判断

---

## 3.9 流程 8: 数据预处理流程 (Data Preprocessing Pipeline)

### 3.9.1 流程图

```mermaid
flowchart TD
    START(["Harry_Potter_Book_1.pdf"]) --> A["split_into_chapters(pdf_path)"]
    
    A --> B["PyPDF2.PdfReader 读取全部页"]
    B --> C["拼接全文文本"]
    C --> D["正则分割: re.split('CHAPTER [A-Z]+)', text)"]
    D --> E["生成 17 个章节 Document<br/>metadata={'chapter': N}"]
    
    E --> F["replace_t_with_space(chapters)<br/>替换制表符"]
    
    F --> G{"并行处理"}
    
    G --> H["路径 1: 章节摘要"]
    H --> H1["create_chapter_summary(chapter)"]
    H1 --> H1a["计算 token 数"]
    H1a --> H1b{"token < 16000?"}
    H1b -->|"是"| H1c["stuff chain"]
    H1b -->|"否"| H1d["map_reduce chain"]
    H1c & H1d --> H2["生成摘要 Document"]
    H2 --> H3["encode_chapter_summaries()"]
    H3 --> H4["chapter_summaries_vector_store"]
    
    G --> I["路径 2: 引文提取"]
    I --> I1["extract_book_quotes_as_documents(documents)"]
    I1 --> I2["正则提取: '至少 50 字符的引号'"]
    I2 --> I3["encode_quotes()"]
    I3 --> I4["book_quotes_vectorstore"]
    
    G --> J["路径 3: 文本分块"]
    J --> J1["RecursiveCharacterTextSplitter<br/>chunk_size=1000, overlap=200"]
    J1 --> J2["~700 个文本块"]
    J2 --> J3["encode_book()"]
    J3 --> J4["chunks_vector_store"]
    
    H4 & I4 & J4 --> END(["3 个 FAISS 索引持久化"])
    
    style A fill:#e8f5e9
    style H1 fill:#e3f2fd
    style I1 fill:#fff3e0
    style J1 fill:#f3e5f5
```

### 3.9.2 详细步骤说明

**章节分割** (`helper_functions.py:59-89`):
- 使用正则 `r'(CHAPTER\s[A-Z]+(?:\s[A-Z]+)*)'` 分割
- 生成 Document 对象，metadata 包含章节号

**章节摘要** (Notebook Cell 16):
- Token 计数决定使用 `stuff` 还是 `map_reduce`
- GPT-3.5-turbo 生成摘要（成本优化）

**引文提取** (`helper_functions.py:92-105`):
- 正则: `r'“(.{50,?)”'` 匹配中文引号内的长文本
- 过滤短引文（min_length=50）

---

## 3.10 流程 9: 评估流程 (Evaluation Pipeline)

### 3.10.1 流程图

```mermaid
flowchart TD
    START(["预定义问题列表 (4 个)"]) --> A["遍历每个问题"]
    
    A --> B["execute_plan_and_print_steps(input)"]
    B --> C["执行完整 Agent 流程"]
    C --> D["收集 final_answer"]
    D --> E["收集 aggregated_context"]
    
    E --> F["构造 data_samples dict"]
    F --> F1["questions: List[str]"]
    F2["answers: List[str]"]
    F3["contexts: List[List[str]]"]
    F4["ground_truth: List[str]"]
    
    F1 & F2 & F3 & F4 --> G["Dataset.from_dict(data_samples)"]
    
    G --> H["Ragas evaluate()"]
    H --> H1["answer_correctness"]
    H2["faithfulness"]
    H3["answer_relevancy"]
    H4["context_recall"]
    H5["answer_similarity"]
    
    H1 & H2 & H3 & H4 & H5 --> I["results_df"]
    I --> J["analyse_metric_results(results_df)"]
    J --> END(["打印 5 项指标分数"])
    
    style H fill:#e3f2fd
    style J fill:#e8f5e9
```

### 3.10.2 评估指标说明

| 指标 | 定义 | 计算方式 | 理想值 |
|------|------|----------|--------|
| **faithfulness** | 答案是否基于检索文档 | NLI 模型判断 | 1.0 |
| **answer_relevancy** | 答案与问题的相关性 | 嵌入相似度 | 1.0 |
| **context_precision** | 检索文档的相关比例 | 人工/模型标注 | 1.0 |
| **context_relevancy** | 检索文档与问题的相关性 | 句子级判断 | 1.0 |
| **answer_similarity** | 答案与 ground truth 的语义相似度 | 嵌入相似度 | 1.0 |
| **answer_correctness** | 答案的事实正确性 | 综合判断 | 1.0 |

---

## 3.11 流程 10: Streamlit 可视化流程

### 3.11.1 流程图

```mermaid
flowchart TD
    START(["用户点击 'Run Agent'"]) --> A["execute_plan_and_print_steps()"]
    
    A --> B["创建 config = {'recursion_limit': 45}"]
    B --> C["plan_and_execute_app.stream(inputs, config)"]
    
    C --> D["创建 progress_bar"]
    D --> E["遍历 stream 输出"]
    
    E --> F["update_placeholders_and_graph()"]
    F --> F1["创建 pyvis Network 图<br/>(当前节点绿色高亮)"]
    F1 --> F2["write_html → tempfile"]
    F2 --> F3["components.html 嵌入 Streamlit"]
    
    F --> G["检测状态变化"]
    G --> H{"current_state != previous_state?"}
    H -->|"是"| H1["更新占位符<br/>(plan, past_steps, aggregated_context)"]
    H -->|"否"| H2["仅更新图可视化"]
    
    H1 --> I["更新 progress_bar"]
    H2 --> I
    
    I --> J{"step >= recursion_limit?"}
    J -->|"否"| E
    J -->|"是"| K["输出最终 response"]
    
    K --> END(["显示 Final Answer"])
    
    style F1 fill:#e8f5e9
    style H fill:#f3e5f5
```

### 3.11.2 可视化组件架构

```mermaid
flowchart LR
    subgraph StreamlitLayout["Streamlit 布局"]
        direction TB
        Title["标题: Real-Time Agent Execution Visualization"]
        Input["输入框 + Run 按钮"]
        Graph["图可视化区域 (pyvis)"]
        
        subgraph Columns["三列布局"]
            Col1["Plan 列"]
            Col2["Past Steps 列"]
            Col3["Aggregated Context 列"]
        end
        
        Answer["Final Answer 显示"]
    end
```

---

## 3.12 流程 11: 异常处理与边界条件

### 3.12.1 异常处理流程

```mermaid
flowchart TD
    START(["开始执行"]) --> A{"输入验证"}
    
    A -->|"空问题"| A1["Streamlit 显示错误"]
    A -->|"有效问题"| B["进入主图"]
    
    B --> C{"LLM 调用"}
    C -->|"超时/网络错误"| C1["LangChain 自动重试<br/>(默认 3 次)"]
    C1 --> C2{"重试成功?"}
    C2 -->|"否"| C3["抛出 Exception<br/>Streamlit 显示错误"]
    C2 -->|"是"| D["继续执行"]
    
    C -->|"成功"| D
    
    D --> E{"递归深度"}
    E -->|"达到 recursion_limit"| E1["终止执行<br/>返回已聚合信息"]
    E -->|"未达限制"| F["继续循环"]
    
    F --> G{"工具选择"}
    G -->|"无效工具"| G1["raise ValueError<br/>终止执行"]
    G -->|"有效工具"| H["执行工具"]
    
    H --> I{"子图循环"}
    I -->|"蒸馏验证循环超过 N 次"| I1["可能无限循环<br/>(无显式保护)"]
    I -->|"正常结束"| J["返回结果"]
```

### 3.12.2 边界条件清单

| 边界条件 | 当前处理 | 风险 | 建议 |
|----------|----------|------|------|
| **空问题** | 无显式检查 | LLM 可能返回无效计划 | 增加输入校验 |
| **LLM 超时** | LangChain 自动重试 | 长时间等待 | 设置超时 + 降级 |
| **递归过深** | `recursion_limit=45` | 可能过早终止 | 动态调整限制 |
| **蒸馏循环** | 无显式限制 | 可能无限循环 | 增加最大循环次数 |
| **空检索结果** | 无显式检查 | 空上下文导致 LLM 幻觉 | 增加空结果检测 |
| **上下文溢出** | 无显式检查 | 超出 LLM 上下文窗口 | 增加上下文截断 |
| **无效工具** | raise ValueError | 执行终止 | 默认回退到 Tool A |

---

## 3.13 本章小结

本章详细描述了系统的 **11 个核心业务流程**：

1. **端到端主流程**: 完整的 Phase 1→2→3 流程
2. **问题匿名化**: 消除 LLM 先验偏见
3. **计划生成与细化**: 从问题到可执行任务
4. **任务路由**: 动态选择检索工具或回答
5. **检索子图**: 检索-蒸馏-验证闭环
6. **回答子图**: CoT 推理 + 幻觉检测
7. **重规划**: 动态更新执行计划
8. **数据预处理**: PDF → 3 路向量存储
9. **评估流程**: Ragas 多维度质量评估
10. **Streamlit 可视化**: 实时图状态展示
11. **异常处理**: 边界条件与容错

**下一章**: [04-module-structure.md](./04-module-structure.md) — 深入分析模块结构与依赖关系。

---

☕️ 制作不易，请我喝咖啡☕️关注我➕