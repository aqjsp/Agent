# 第5章：Harness 核心机制 —— Planning

> **本章你将理解：** 任务分解的原理和陷阱、Plan-and-Execute 与 ReAct 的对比、自反思与自我纠错机制。
>
> **前置知识：** 第1-4章（Agent 循环、Function Calling、工具系统、记忆系统）。
>
> **学完这章你能做：** 为 Agent 选择合适的规划模式、实现 Plan-and-Execute Agent、诊断规划失败的问题。

前面三章解决了 Agent 的"动手"和"记住"能力。但遇到复杂任务——"帮我调研 AIGC 在教育领域的应用，写一份 3000 字报告"——Agent 还需要"想清楚再做"的能力。这就是规划。

规划是把"帮我做X"变成一系列可执行步骤的能力。没有规划的 Agent 只能被动响应，遇到复杂任务就迷失方向——要么跳过关键步骤，要么在错误的方向上越走越远。

---

## 1 任务分解：从目标到可执行步骤

LLM 天然擅长分解任务——你给它一个目标，它能列出子步骤。但"列出步骤"和"正确执行步骤"是两件事。

### LLM 做任务分解的三个陷阱

![](../image/ch5_task_decomposition_traps.svg)

**陷阱一：步骤粒度不一致。** LLM 可能把"搜索资料"作为一个步骤（太粗），同时把"打开浏览器、输入关键词、点击搜索"作为三个步骤（太细）。正确粒度是每个步骤对应一次工具调用。

**陷阱二：忽略依赖关系。** LLM 可能列出"1. 写总结 2. 搜索资料"——顺序反了。搜索是写总结的前置条件。

**陷阱三：幻觉出不存在的能力。** LLM 可能规划"1. 调用数据库查询"——但你的 Agent 根本没有数据库工具。它规划的是自己想象中的能力，不是实际拥有的。

### 应对：让 LLM 知道自己有什么

陷阱三的根因是 LLM 规划时不知道自己有哪些工具。解决方案：**在规划提示中明确列出可用工具，并给出输出示例。**

```python
PLANNING_PROMPT = """你是一个任务规划助手。根据用户的目标，制定执行计划。

可用工具：
- get_weather(city): 获取天气信息
- search_web(query): 搜索互联网
- calculate(expression): 数学计算

规划规则：
1. 每个步骤必须对应一次工具调用或一次信息整合
2. 有依赖关系的步骤必须按顺序排列
3. 不能使用上面没有列出的工具
4. 步骤数量控制在3-7步

请输出JSON格式的计划：
{"steps": [{"action": "工具调用或整合", "tool": "工具名或null", "args": {"参数名": "参数值"}, "reason": "为什么需要这步"}]}

输出示例：
{"steps": [
    {"action": "获取北京天气", "tool": "get_weather", "args": {"city": "北京"}, "reason": "需要北京实时天气数据"},
    {"action": "获取上海天气", "tool": "get_weather", "args": {"city": "上海"}, "reason": "需要上海实时天气数据"},
    {"action": "对比两地户外运动条件", "tool": null, "args": {}, "reason": "综合分析两地天气数据"}
]}"""
```

**输出示例的关键作用：** 没有示例时，LLM 输出的 JSON 格式经常不稳定——`action` 字段可能写成 `步骤`，`tool` 可能写成 `tool_name`。加一个示例，格式准确率大幅提升。这是第2章 CoT 原理的直接应用——给模型一个"锚点"，让它知道期望的输出长什么样。

### 好的计划长什么样

一个合格的计划有三个特征：

**特征一：每个步骤有明确的完成标准。** "搜索资料"不是好步骤——什么时候算搜完了？"搜索 3 篇关于 X 的文章"才是。完成标准让 Agent 知道何时进入下一步。

**特征二：步骤之间没有歧义的依赖。** 明确标注"步骤 3 依赖步骤 1 和 2 的结果"。如果 LLM 规划时没有标注依赖，Harness 层应该按顺序执行——假设前后有依赖。

**特征三：有回退路径。** 如果步骤 2 执行失败怎么办？继续步骤 3？还是重新规划？好的计划应该考虑失败——至少在最关键的步骤上标注备选方案。

对比两个计划：

```
# 差的计划
1. 搜索资料
2. 分析数据
3. 写报告

# 好的计划
1. search_web("AIGC教育应用 2024") → 获取至少3篇文章摘要
2. 整理发现：按"应用场景/技术方案/挑战"分类（依赖步骤1的结果）
3. 撰写报告结构：摘要+3个案例+结论（依赖步骤2的结果）
   备选：如果步骤1搜索结果不足，换关键词重搜
```

好的计划不一定更长，但更精确。**精确的计划让 Harness 层有据可查——"Agent 偏离了吗？步骤完成了吗？需要重规划吗？"**

---

## 2 两种规划模式：ReAct vs Plan-and-Execute

![](../image/ch5_react_vs_plan_execute.svg)

### ReAct：边做边想

ReAct 是前面所有章节使用的模式——每一步都是"先思考，再行动，再观察结果"。规划是隐式的、逐步的。

```
用户: "北京和上海哪个更适合举办户外活动？"

ReAct 过程：
1. [Thought] 需要两地天气 → get_weather(北京) + get_weather(上海)
2. [Thought] 还需要空气质量 → search_web("北京上海空气质量对比")
3. [Thought] 信息够了 → 综合分析，给出建议
```

ReAct 的优点是灵活——每一步的决策基于前一步的结果，能根据新信息调整方向。缺点是缺乏全局视角——Agent 不知道最终需要几步，可能在第 2 步就走偏了。

### Plan-and-Execute：先规划再执行

Plan-and-Execute 把规划和执行分成两个阶段：先用 LLM 生成完整计划，再逐步执行。

```
用户: "北京和上海哪个更适合举办户外活动？"

Plan 阶段：
1. 获取北京当前天气和空气质量
2. 获取上海当前天气和空气质量
3. 对比两地的户外活动条件（温度、降水、空气质量）
4. 给出建议

Execute 阶段：
Step 1: get_weather(北京) + search_web("北京空气质量")
Step 2: get_weather(上海) + search_web("上海空气质量")
Step 3: [LLM 对比分析]
Step 4: [LLM 输出建议]
```

Plan-and-Execute 的优点是有全局视角——先想清楚再做，不容易走偏。缺点是不够灵活——如果执行中发现计划有误，需要重新规划。

### 两种模式的适用场景

| 维度 | ReAct | Plan-and-Execute |
|------|-------|-----------------|
| 步骤数 | 少（1-5步） | 多（5+步） |
| 不确定性 | 高（中间结果可能改变方向） | 低（路径相对确定） |
| 错误恢复 | 自然（下一步自动调整） | 需要显式重规划 |
| Token 消耗 | 较低（逐步推理） | 较高（规划 + 执行两次调用） |
| 适用任务 | 简单查询、单工具调用 | 研究报告、多步分析 |

**实用建议：** 不确定用哪个时，先用 ReAct。ReAct 跑不通了（步骤太多、迷失方向），再加 Plan-and-Execute。不要一开始就用 Plan-and-Execute——大多数任务 ReAct 就够了。

---

## 3 Plan-and-Execute 的实现

手动实现 Plan-and-Execute Agent。核心是四个函数：规划、执行、判断完成、生成回答，用循环串起来。第6章会用 LangGraph 的 StateGraph 重新实现 Supervisor 模式——这里先理解原理。

![](../image/ch5_plan_execute_flow.svg)

```python
import json
from openai import OpenAI
from pydantic import BaseModel

# 复用第3章的工具注册系统
# from ch3 import get_tool_map, get_tools_schema, _tool_registry

client = OpenAI()

# ---- 数据模型 ----

class Step(BaseModel):
    action: str
    tool: str | None = None
    args: dict = {}
    reason: str

class Plan(BaseModel):
    steps: list[Step]

# ---- 规划节点 ----

def plan_step(state: dict) -> dict:
    """根据用户目标生成执行计划。"""
    user_goal = state["user_goal"]
    executed = state.get("executed_steps", [])
    results = state.get("step_results", [])

    # 如果有已执行的步骤，把结果也告诉 LLM，让它调整计划
    context = ""
    if executed:
        context = f"\n\n已完成的步骤和结果：\n"
        for step, result in zip(executed, results):
            context += f"- {step.action}: {result}\n"

    response = client.chat.completions.create(
        model="gpt-4o",
        response_format={"type": "json_object"},
        messages=[
            {"role": "system", "content": PLANNING_PROMPT},
            {"role": "user", "content": f"用户目标：{user_goal}{context}"}
        ]
    )

    plan_data = json.loads(response.choices[0].message.content)
    plan = Plan(steps=[Step(**s) for s in plan_data["steps"]])

    return {"plan": [s.model_dump() for s in plan.steps]}

# ---- 执行节点 ----

def execute_step(state: dict) -> dict:
    """执行计划中的下一步。"""
    plan = state["plan"]
    executed = state.get("executed_steps", [])
    step_idx = len(executed)

    if step_idx >= len(plan):
        return {}

    step = Step(**plan[step_idx])
    result = ""

    if step.tool and step.tool in get_tool_map():
        func = get_tool_map()[step.tool]
        try:
            result = func(**step.args)
            if isinstance(result, dict):
                result = json.dumps(result, ensure_ascii=False)
        except Exception as e:
            result = f"执行失败: {e}"
    else:
        # 没有工具的步骤，让 LLM 处理——传入之前的结果作为上下文
        prev_results = state.get("step_results", [])
        context = ""
        if prev_results:
            context = "\n\n之前步骤的结果：\n" + "\n".join(
                f"步骤{i+1}: {r[:500]}" for i, r in enumerate(prev_results)
            )
        result = client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[
                {"role": "system", "content": "根据已有信息完成以下任务，输出简洁结果。"},
                {"role": "user", "content": f"任务：{step.action}{context}"}
            ]
        ).choices[0].message.content

    executed.append(step.model_dump())
    results = state.get("step_results", [])
    results.append(result)

    return {"executed_steps": executed, "step_results": results}

# ---- 判断是否完成 ----

def should_continue(state: dict) -> str:
    """判断是继续执行还是输出最终回答。"""
    plan = state.get("plan", [])
    executed = state.get("executed_steps", [])

    if len(executed) >= len(plan):
        return "respond"
    return "execute"

# ---- 最终回答节点 ----

def respond(state: dict) -> dict:
    """根据所有步骤的结果生成最终回答。"""
    user_goal = state["user_goal"]
    results = state.get("step_results", [])

    summary = "\n".join(f"步骤{i+1}结果: {r}" for i, r in enumerate(results))

    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": "根据以下步骤的执行结果，回答用户的原始问题。回答要完整、有逻辑。"},
            {"role": "user", "content": f"用户目标：{user_goal}\n\n执行结果：\n{summary}"}
        ]
    )

    return {"response": response.choices[0].message.content}
```

运行效果：

```python
state = {
    "user_goal": "对比北京和上海今天的天气，哪个更适合户外运动？",
    "plan": [],
    "executed_steps": [],
    "step_results": []
}

# 先规划
state.update(plan_step(state))
print("计划:", [s["action"] for s in state["plan"]])
# → ['获取北京天气', '获取上海天气', '对比两地户外运动条件', '给出建议']

# 逐步执行（加上迭代上限，防止异常计划导致死循环）
max_steps = len(state.get("plan", [])) + 3  # 允许重规划后多几步
step_count = 0
while should_continue(state) == "execute" and step_count < max_steps:
    state.update(execute_step(state))
    step_count += 1

# 生成回答
state.update(respond(state))
print(state["response"])
```

---

## 4 自反思与自我纠错

规划不是一次就对的。执行中可能发现：搜索结果跟预期不符、某个步骤执行失败、信息不足以得出结论。这时候需要自反思——让 Agent 检查自己的输出，发现问题并修正。

![](../image/ch5_self_refine_vs_reflexion.svg)

### Self-Refine

Self-Refine 的思路简单：生成初稿 → 自我批评 → 改进 → 再批评 → ... 直到满意或达到次数上限。

```python
def self_refine(task: str, max_rounds: int = 3) -> str:
    """自反思式改进。"""
    # 生成初稿
    draft = client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": "你是一个严谨的研究助手。"},
            {"role": "user", "content": task}
        ]
    ).choices[0].message.content

    for i in range(max_rounds):
        # 自我批评
        critique = client.chat.completions.create(
            model="gpt-4o",
            messages=[
                {"role": "system", "content": "你是一个严格的审稿人。指出以下回答中的问题：事实错误、逻辑漏洞、遗漏要点、表达不清。如果没有问题，输出 APPROVED。"},
                {"role": "user", "content": draft}
            ]
        ).choices[0].message.content

        if "APPROVED" in critique:
            break

        # 改进
        draft = client.chat.completions.create(
            model="gpt-4o",
            messages=[
                {"role": "system", "content": "你是一个严谨的研究助手。根据审稿意见改进你的回答。"},
                {"role": "user", "content": f"原始回答：\n{draft}\n\n审稿意见：\n{critique}\n\n请改进。"}
            ]
        ).choices[0].message.content

    return draft
```

### Reflexion

Reflexion 比 Self-Refine 多了一步：**把反思结果存入记忆，让下次遇到类似任务时能避免同样的错误。**

```python
def reflexion_agent(task: str, max_attempts: int = 3) -> str:
    """带记忆的反思式 Agent。"""
    reflections = []  # 存储反思结果

    for attempt in range(max_attempts):
        # 执行任务（带上之前的反思）
        reflection_context = ""
        if reflections:
            reflection_context = "\n\n之前的经验教训：\n" + "\n".join(reflections)

        result = client.chat.completions.create(
            model="gpt-4o",
            messages=[
                {"role": "system", "content": "你是一个会从错误中学习的助手。" + reflection_context},
                {"role": "user", "content": task}
            ]
        ).choices[0].message.content

        # 评估结果（用结构化输出确保格式可靠）
        evaluation = client.chat.completions.create(
            model="gpt-4o",
            response_format={"type": "json_object"},
            messages=[
                {"role": "system", "content": '评估以下回答的质量。输出JSON: {"score": 分数(1-10的整数), "issues": "具体问题"}'},
                {"role": "user", "content": f"任务：{task}\n\n回答：{result}"}
            ]
        ).choices[0].message.content

        eval_data = json.loads(evaluation)
        if eval_data.get("score", 0) >= 8:
            return result  # 质量够好

        # 反思：为什么不好？下次怎么改进？
        reflection = client.chat.completions.create(
            model="gpt-4o",
            messages=[
                {"role": "system", "content": "分析失败原因，总结经验教训。输出一条具体的改进建议。"},
                {"role": "user", "content": f"任务：{task}\n回答：{result}\n评估：{evaluation}"}
            ]
        ).choices[0].message.content

        reflections.append(reflection)

    return result
```

Self-Refine 和 Reflexion 的代价是多次 LLM 调用。只在质量要求高的场景使用——比如生成报告、代码审查。简单查询用 ReAct 就够了。

---

## 5 规划失败的模式

即使有了规划，Agent 还是会失败。三种常见的失败模式：

![](../image/ch5_planning_failures.svg)

### 过度规划

Agent 把简单任务搞复杂了。用户问"今天北京天气"，Agent 规划了5步：查天气、查空气质量、查紫外线、查花粉、综合分析。用户只需要一个温度。

**应对：** 在规划提示中加约束——"步骤数量控制在3-7步。如果任务简单，1-2步就够。"

### 规划偏离

执行过程中逐步偏离原始目标。用户要"调研AIGC在教育领域的应用"，Agent 搜索了"AI教育工具"，然后对某个工具做了深度分析，最后输出了一份产品评测——偏离了"调研"这个目标。

**应对：** 在每一步执行后，检查当前步骤是否还在为原始目标服务。可以加一个"目标检查"节点：

```python
def goal_check(state: dict) -> dict:
    """检查当前执行是否偏离目标。"""
    user_goal = state["user_goal"]
    recent_result = state["step_results"][-1] if state["step_results"] else ""

    check = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content": "判断最新的执行结果是否还在服务于用户的目标。输出 ON_TRACK 或 OFF_TRACK。"},
            {"role": "user", "content": f"用户目标：{user_goal}\n最新结果：{recent_result[:500]}"}
        ]
    ).choices[0].message.content

    if "OFF_TRACK" in check:
        return {"needs_replan": True}
    return {"needs_replan": False}
```

**goal_check 的成本考量：** 每一步都调用 goal_check 会增加一次 LLM 调用，token 消耗增加约 30%。实际项目中不需要每步都检查——只在步骤数超过 5 步或执行时间超过预期时才触发。

### 死循环

Agent 在某个步骤反复执行，一直得不到满意的结果。比如搜索了5次都没找到想要的信息，每次都换个关键词再搜。

**应对：** `max_iterations` 是硬限制。Plan-and-Execute 中还要加"单步重试上限"——某个步骤失败超过2次，标记为失败并跳过。

---

## 6 动手实验

### 实验一：同一任务，两种模式对比

用同一个任务："调研 Python 和 Rust 在后端开发中的优劣，给出选型建议"。

1. 用 ReAct 模式跑，记录步骤数和最终输出
2. 用 Plan-and-Execute 模式跑，记录步骤数和最终输出
3. 对比：哪个更全面？哪个更高效？哪个跑偏了？

### 实验二：触发自反思

1. 让 Agent 写一段代码（比如排序算法）
2. 用 Self-Refine 让它自我批评和改进
3. 对比初稿和最终版本的质量差异

### 实验三：触发规划失败

1. 问一个简单问题（"1+1等于几"），观察 Agent 是否过度规划
2. 问一个需要多次搜索的问题（"比较三家云服务商的价格"），观察是否出现规划偏离
3. 给一个搜索结果很少的冷门话题，观察是否死循环

---

下一章进入 Multi-Agent 协作。当单个 Agent 的上下文窗口不够用、角色太多导致混淆、或工具太多导致选择准确率下降时，就需要把一个 Agent 拆成多个。
