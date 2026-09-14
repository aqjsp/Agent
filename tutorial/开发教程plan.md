Agent = Model + Harness 开发教程
将完整教程写入 开发教程.md，结构如下：

教程结构
第1章：Agent 本质 —— Model + Harness
什么是 Agent，与普通 LLM 调用的本质区别
Model（模型层）：LLM 的能力边界，为什么 Model 单独不够
Harness（脚手架层）：编排、工具、记忆、规划 —— 让模型"动起来"的基础设施
用 svg 图展示 Agent 架构：Model 作为大脑，Harness 作为四肢和感官
第2章：Model 深度解析
LLM 推理的本质：next-token prediction → Chain of Thought
System Prompt 的工程化设计
Function Calling / Tool Use 协议（OpenAI 兼容格式）
代码示例：用 OpenAI Python SDK 发起一次带 tools 的调用
第3章：Harness 核心机制 —— Tool Use
Tool 的定义、注册与发现
ReAct 循环：Thought → Action → Observation
LangChain Tool 抽象 + LangGraph 的 tool_node
代码示例：手写一个 ReAct Agent，能搜索网页 + 执行 Python 代码
第4章：Harness 核心机制 —— Memory
短期记忆 vs 长期记忆
对话记忆：Buffer / Summary / Sliding Window
长期记忆：向量数据库 + RAG 检索
LangGraph 的 checkpointer 持久化机制
代码示例：带记忆的对话 Agent，跨 session 记住用户偏好
第5章：Harness 核心机制 —— Planning
任务分解：从目标到可执行步骤
Plan-and-Execute 模式 vs ReAct 模式对比
自反思与自我纠错（Self-Refine, Reflexion）
代码示例：LangGraph 实现 Plan-and-Execute Agent
第6章：Multi-Agent 协作
为什么需要多 Agent：单一 Agent 的能力瓶颈
协作模式：Router / Supervisor / Swarm / Hierarchical
LangGraph 的 sub-graph 机制
代码示例：Supervisor 模式的多 Agent 系统（Researcher + Coder + Reviewer）
第7章：实战项目 —— 端到端 Agent
整合所有模块，构建一个完整的 Research Agent
能搜索、能记忆、能规划、多角色协作
生产级考量：错误处理、成本控制、可观测性
实现方式
将全部内容写入 开发教程.md
每章包含：概念讲解 → 架构图（Mermaid）→ 完整可运行代码
代码基于 Python 3.11+ / LangChain 0.3+ / LangGraph 0.2+