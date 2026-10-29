# Agent

除 Model 之外，Agent 落地要做的事。面向后端开发者。阅读顺序在 [00-map.md](00-map.md)。

网站阅读版：[aqjszz.com/agent](https://aqjszz.com/agent/)。

## 目录

```text
agents/         模型、Harness、怎么把循环跑起来
running/        一次 turn 怎么走完、怎么停、崩溃后怎么续
tools/          一次调用从说出口到执行完
handoffs/       再开一个循环：控制权、一致性、通信、A2A
guardrails/     注入和权限
sessions/       记忆、State、窗口里放什么
tracing/        可观测、评估、成本
mcp/            工具在运行时被发现
hitl/           人的等待怎么挂在会话上
sandbox/        模型写的代码跑在哪
interview/      面试题
image/          配图
```
