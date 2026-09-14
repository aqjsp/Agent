# 第6章：Multi-Agent 协作

> **本章你将理解：** 为什么需要多 Agent、四种协作模式的原理和适用场景、Agent 间通信机制、LangGraph 的多 Agent 实现。
>
> **前置知识：** 第1-5章（Agent 循环、工具系统、记忆系统、规划机制）。
>
> **学完这章你能做：** 判断何时需要多 Agent、选择合适的协作模式、实现 Supervisor 模式的多 Agent 系统。

一个 Agent 能做的事有限。不是能力不够，是三个硬限制：

1. **上下文窗口有限。** 给一个 Agent 塞 20 个工具 + 完整对话历史，128K tokens 很快就满了。
2. **角色混淆。** 让同一个 Agent 既搜索资料又写代码又审查代码，它容易角色错乱——搜索时想着代码怎么写，写代码时忘了搜索结果。
3. **工具冲突。** 搜索工具和代码执行工具的 description 可能重叠，Agent 选错工具的概率上升。

多 Agent 不是"更多能力"——是"更清晰的分工"。每个 Agent 只做一件事，上下文干净，角色明确，工具少而精。

---

## 1 四种协作模式

### Router 模式

```
用户 → Router Agent → Agent A（天气）
                   → Agent B（搜索）
                   → Agent C（计算）
```

Router 只做一件事：判断用户意图，分发给对应的专业 Agent。最简单的多 Agent 模式。

**适用场景：** 意图分类明确、各 Agent 之间没有依赖关系。客服系统、知识库问答。

**缺点：** Router 选错了就全错了。没有纠错机制——一旦分发出去，结果直接返回用户。

Router 的实现非常简单——本质上就是第2章的 `tool_choice` 指定工具的泛化版本：

```python
def router(user_message: str) -> str:
    """根据用户意图路由到对应 Agent。"""
    response = client.chat.completions.create(
        model="gpt-4o-mini",  # 路由用便宜模型就够了
        response_format={"type": "json_object"},
        messages=[
            {"role": "system", "content": """判断用户意图，输出 JSON: {"intent": "weather|search|calculate|unknown"}

意图分类规则：
- weather: 用户询问天气、温度、是否需要带伞等
- search: 用户需要搜索信息、查找资料、了解最新动态
- calculate: 用户需要数学计算、数据统计
- unknown: 无法判断或不在以上类别"""},
            {"role": "user", "content": user_message}
        ]
    )
    intent = json.loads(response.choices[0].message.content).get("intent", "unknown")
    return intent

# 路由分发
agent_map = {
    "weather": weather_agent,
    "search": search_agent,
    "calculate": calculate_agent,
}

def route_and_execute(user_message: str) -> str:
    intent = router(user_message)
    if intent == "unknown" or intent not in agent_map:
        return "抱歉，我无法处理这个问题。我可以帮你查天气、搜索信息、或做计算。"
    return agent_map[intent](user_message)
```

Router 模式最大的风险是**分类错误无法自纠**。缓解方法：在 `unknown` 分支返回兜底回答，而不是随意分发。如果分类准确率要求高，可以把 Router 改成"Router + 验证"模式——Router 分发后，检查 Agent 输出是否合理，不合理则回退到通用 Agent。

### Supervisor 模式

```
用户 → Supervisor → Worker A（搜索）→ 结果回 Supervisor
                  → Worker B（写代码）→ 结果回 Supervisor
                  → Worker C（审查）→ 结果回 Supervisor
                  → 最终回答
```

Supervisor 做三件事：拆分任务、分配给 Worker、整合结果。和 Router 的区别是：Supervisor 会看 Worker 的输出，决定下一步——再分配给另一个 Worker，还是直接回答。

**适用场景：** 多步骤任务、Worker 之间有依赖关系。研究报告、代码开发。

**缺点：** Supervisor 是单点——它自己的上下文窗口会随着 Worker 输出的积累而增长。

### Swarm 模式

```
Agent A ←→ Agent B ←→ Agent C
  ↓              ↓           ↓
完成时交出控制权给最合适的 Agent
```

Swarm 没有中心节点。每个 Agent 可以把控制权交给另一个 Agent。OpenAI 的 Swarm 框架就是这种模式。

**适用场景：** Agent 之间是对等关系、不需要全局协调。客户服务流转。

**缺点：** 可能出现 Agent 之间来回踢皮球——A 交给 B，B 又交给 A。

### Hierarchical 模式

```
         Top Supervisor
        /              \
  Mid Supervisor    Mid Supervisor
   /      \           /      \
Agent A  Agent B   Agent C  Agent D
```

多层级管理。顶层 Supervisor 做战略决策，中层做战术分配，底层执行。

**适用场景：** 大型复杂项目。软件工程项目（架构师 → 模块负责人 → 开发者）。

**缺点：** 复杂度最高，调试最难。层数超过 3 层后，上下文在传递过程中损失严重。

### 模式选择

| 模式 | 复杂度 | 适用场景 | Agent 数量 |
|------|--------|---------|-----------|
| Router | 低 | 意图分类 | 3-10 |
| Supervisor | 中 | 多步骤任务 | 2-5 |
| Swarm | 中 | 对等协作 | 2-5 |
| Hierarchical | 高 | 大型项目 | 5+ |

**实用建议：** 从 Supervisor 开始。Router 太简单，Swarm 和 Hierarchical 太复杂。Supervisor 是最常用的模式，也是理解其他模式的基础。

---

## 2 Agent 间通信

多 Agent 系统的核心问题：Agent 之间怎么传递信息？两种方式：

### 消息传递（Message Passing）

每个 Agent 有自己的 messages 列表。Agent A 的输出作为 Agent B 的输入——通过函数调用传递，不共享状态。

```python
def researcher_agent(query: str) -> str:
    """研究 Agent：搜索并整理信息。"""
    # 有自己的 messages 列表
    messages = [
        {"role": "system", "content": "你是研究助手。搜索并整理信息，输出结构化的研究结果。"},
        {"role": "user", "content": query}
    ]
    # ... ReAct 循环 ...
    return result

def writer_agent(research_result: str) -> str:
    """写作 Agent：基于研究结果写报告。"""
    # 有自己的 messages 列表
    messages = [
        {"role": "system", "content": "你是技术写作助手。基于研究结果撰写清晰的报告。"},
        {"role": "user", "content": f"请基于以下研究结果写报告：\n{research_result}"}
    ]
    # ... ReAct 循环 ...
    return result

# 串行调用
research = researcher_agent("AIGC在教育领域的应用")
report = writer_agent(research)
```

优点：每个 Agent 的上下文干净——只看到跟自己任务相关的信息。不会出现角色混淆。

缺点：信息在传递中可能损失。Researcher 找到了 20 条信息，但总结成 500 字传给 Writer——Writer 只能看到这 500 字。

### 共享状态（Shared State）

所有 Agent 共享一个状态对象。LangGraph 的 `StateGraph` 就是这种模式。

```python
from typing import TypedDict, Annotated
from langgraph.graph import StateGraph, END

class ResearchState(TypedDict):
    query: str
    search_results: list[str]
    draft: str
    review_comments: list[str]
    final_report: str

def researcher(state: ResearchState) -> dict:
    """研究 Agent：搜索并把结果写入共享状态。"""
    results = search_and_organize(state["query"])
    return {"search_results": results}

def writer(state: ResearchState) -> dict:
    """写作 Agent：从共享状态读取研究结果，写入草稿。"""
    draft = write_draft(state["search_results"])
    return {"draft": draft}

def reviewer(state: ResearchState) -> dict:
    """审查 Agent：从共享状态读取草稿，写入审查意见。"""
    comments = review_draft(state["draft"])
    return {"review_comments": comments}
```

优点：信息不损失——所有 Agent 都能访问完整状态。

缺点：状态对象会越来越大。如果 5 个 Agent 都往状态里写数据，后面的 Agent 要处理的信息量可能超过上下文窗口。

### 选择原则

- Agent 之间是**串行关系**（A 完成后 B 才开始）：消息传递。简单、上下文干净。
- Agent 之间需要**共享中间结果**（A 和 B 都需要 C 的输出）：共享状态。避免重复计算。
- 混合场景：Supervisor 用共享状态管理全局进度，Worker 之间用消息传递。

---

## 3 Supervisor 模式的完整实现

用 LangGraph 实现 Supervisor 模式（Researcher + Writer + Reviewer）：

```python
import json
from typing import TypedDict
from openai import OpenAI
from langgraph.graph import StateGraph, END

# 复用第3章的工具注册系统
# from ch3 import get_tool_map, get_tools_schema

client = OpenAI()

# ---- 共享状态 ----

class AgentState(TypedDict):
    query: str
    research_result: str
    draft: str
    review_comments: str
    final_output: str
    next_agent: str

# ---- Supervisor ----

SUPERVISOR_PROMPT = """你是一个任务协调者。根据当前状态决定下一步由哪个 Agent 执行。

可选 Agent:
- researcher: 搜索和整理信息
- writer: 撰写报告
- reviewer: 审查报告质量
- FINISH: 任务完成，输出最终结果

当前状态：
- 研究结果: {research_result}
- 草稿: {draft}
- 审查意见: {review_comments}

输出 JSON: {{"next": "agent_name 或 FINISH"}}"""

def supervisor(state: AgentState) -> dict:
    """决定下一步由哪个 Agent 执行。"""
    # 安全地格式化 prompt——只传入安全的字段，避免 state 中的特殊字符破坏格式
    prompt = SUPERVISOR_PROMPT.format(
        research_result=state.get("research_result", "无")[:500],
        draft=state.get("draft", "无")[:500],
        review_comments=state.get("review_comments", "无")[:500]
    )
    response = client.chat.completions.create(
        model="gpt-4o",
        response_format={"type": "json_object"},
        messages=[
            {"role": "system", "content": prompt},
            {"role": "user", "content": state["query"]}
        ]
    )
    decision = json.loads(response.choices[0].message.content)
    return {"next_agent": decision["next"]}

# ---- Researcher ----

def researcher(state: AgentState) -> dict:
    """搜索并整理信息。"""
    # Researcher 只使用搜索类工具
    search_tools = [t for t in get_tools_schema()
                    if t["function"]["name"] in ("search_web", "get_current_date")]

    messages = [
        {"role": "system", "content": "你是研究助手。搜索并整理与用户查询相关的信息。输出结构化的研究结果，包含关键发现和数据。"},
        {"role": "user", "content": state["query"]}
    ]

    # 简化的 ReAct 循环
    for i in range(5):
        response = client.chat.completions.create(
            model="gpt-4o",
            messages=messages,
            tools=search_tools,
            tool_choice="auto"
        )
        msg = response.choices[0].message
        messages.append(msg.model_dump())

        if msg.tool_calls:
            tool_map = get_tool_map()
            for tc in msg.tool_calls:
                func = tool_map.get(tc.function.name)
                if not func:
                    content = f"错误：工具 '{tc.function.name}' 不存在"
                else:
                    try:
                        args = json.loads(tc.function.arguments)
                        result = func(**args)
                        content = json.dumps(result, ensure_ascii=False) if isinstance(result, dict) else str(result)
                    except Exception as e:
                        content = f"执行失败: {e}"
                messages.append({"role": "tool", "tool_call_id": tc.id, "content": content})
        else:
            return {"research_result": msg.content}

    return {"research_result": "研究超时，请重试"}

# ---- Writer ----

def writer(state: AgentState) -> dict:
    """基于研究结果撰写报告。"""
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": "你是技术写作助手。基于研究结果撰写清晰的报告。如有审查意见，请根据意见修改。"},
            {"role": "user", "content": f"研究结果：\n{state['research_result']}\n\n审查意见：\n{state.get('review_comments', '无')}"}
        ]
    )
    return {"draft": response.choices[0].message.content}

# ---- Reviewer ----

def reviewer(state: AgentState) -> dict:
    """审查报告质量。"""
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": "你是严格的审稿人。审查报告的事实准确性、逻辑完整性、表达清晰度。如果质量合格，输出 APPROVED。否则输出具体的修改建议。"},
            {"role": "user", "content": state["draft"]}
        ]
    )
    return {"review_comments": response.choices[0].message.content}

# ---- 路由函数 ----

def route_agent(state: AgentState) -> str:
    """根据 Supervisor 的决策路由到对应 Agent。"""
    next_agent = state.get("next_agent", "researcher")
    if next_agent == "FINISH":
        return "finish"
    return next_agent

# ---- 构建图 ----

builder = StateGraph(AgentState)

# 添加节点
builder.add_node("supervisor", supervisor)
builder.add_node("researcher", researcher)
builder.add_node("writer", writer)
builder.add_node("reviewer", reviewer)

# 添加边
builder.set_entry_point("supervisor")
builder.add_conditional_edges("supervisor", route_agent, {
    "researcher": "researcher",
    "writer": "writer",
    "reviewer": "reviewer",
    "finish": END
})
builder.add_edge("researcher", "supervisor")
builder.add_edge("writer", "supervisor")
builder.add_edge("reviewer", "supervisor")

graph = builder.compile()
```

运行：

```python
result = graph.invoke({"query": "调研 AIGC 在教育领域的应用现状和趋势"})
print(result["draft"])
```

执行流程：

```
supervisor → researcher → supervisor → writer → supervisor → reviewer → supervisor → FINISH
```

Supervisor 在每一步都看当前状态，决定下一步。Reviewer 如果输出 APPROVED，Supervisor 就决定 FINISH。否则让 Writer 根据意见修改。

---

## 4 多 Agent 的工程挑战

### 挑战一：上下文传递的损失

Researcher 找到了 20 条信息，但总结成 500 字传给 Writer。Writer 只能看到 500 字。

**应对：** 关键信息用结构化格式传递（编号列表、关键数据点），而不是一整段叙述。或者用共享状态让 Writer 直接访问原始搜索结果。

结构化传递的示例：

```python
# 差：一大段叙述传给 Writer
"经过搜索，发现 AIGC 在教育领域有三个主要应用：自适应学习、智能辅导和自动评分。其中自适应学习是最成熟的..."

# 好：结构化数据传给 Writer
"""
## 研究发现

### 1. 自适应学习（成熟度：高）
- 来源: [搜索结果1] [搜索结果3]
- 关键数据: 使用自适应学习系统的学生成绩平均提升 15-25%
- 代表产品: Knewton, ALEKS

### 2. 智能辅导（成熟度：中）
- 来源: [搜索结果2]
- 关键数据: 2024 年全球市场规模约 12 亿美元
- 挑战: 情感理解能力不足

### 3. 自动评分（成熟度：中低）
- 来源: [搜索结果4] [搜索结果5]
- 关键数据: 目前主要覆盖选择题和短文，长文评分准确率约 75%
- 争议: 学术界对 AI 评分的伦理争议
"""
```

结构化格式让 Writer 能快速定位需要的信息，而不是在叙述中大海捞针。

### 挑战二：Agent 数量与调试难度

每个 Agent 是独立的 LLM 循环。3 个 Agent 意味着 3 倍的 LLM 调用、3 倍的潜在出错点、3 倍的调试成本。

**应对：** Agent 数量控制在 2-5 个。超过 5 个时考虑合并——两个职责相似的 Agent 应该合并成一个。

### 挑战三：成本

每个 Agent 的每次调用都消耗 tokens。一个 3-Agent 系统处理一个任务可能消耗 3-5 倍于单 Agent 的 tokens。

**应对：** 不同 Agent 用不同模型。Researcher 需要工具调用用 GPT-4o，Writer 不需要工具用 GPT-4o-mini（成本降 10 倍），Reviewer 需要强推理用 GPT-4o。

### 挑战四：死锁

Swarm 模式中，Agent A 把控制权交给 B，B 又交给 A——无限循环。

**应对：** 加全局 `max_handoffs` 限制。和单 Agent 的 `max_iterations` 一样——没有终止条件的循环是 bug。

---

## 5 动手实验

### 实验一：从单 Agent 拆成多 Agent

1. 用第3章的单 Agent 完成一个复杂任务（比如"调研某技术并写一份简报"）
2. 把它拆成 Researcher + Writer 两个 Agent
3. 对比：哪个结果更好？哪个成本更低？

### 实验二：观察 Supervisor 的决策

1. 在 Supervisor 节点打印每次决策和理由
2. 运行一个需要 3 轮以上协作的任务
3. 观察：Supervisor 是按预期路由的吗？有没有把任务分给错误的 Agent？

### 实验三：触发 Reviewer 修改循环

1. 给 Writer 一个很低质量的 System Prompt，让它写出的报告很差
2. 观察 Reviewer 是否能发现问题
3. 观察 Writer 修改后质量是否提升
4. 观察修改循环是否会无限进行（如果没有终止条件）

---

下一章是实战项目——整合所有模块，从零构建一个生产级 Research Agent。工具系统、记忆系统、规划系统、多 Agent 协作，全部用上。
