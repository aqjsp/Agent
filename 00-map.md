# 阅读顺序

十个目录按一次请求经过的部件排：Agents、Running、Tools、Handoffs、Guardrails、Sessions、Tracing、MCP、HITL、Sandbox。进程死了、换了机器、人等了几天还要续上，读 [Runtime：模型之外，Agent 真正活着的那一层](running/全景.md)。可以读完各目录再读，也可以先读它再按部件回头。

面试题不进这条顺序，单独在 [interview/Agent面试题.md](interview/Agent面试题.md)。

## 1. Agents

循环是谁的，模型看见什么，换一套 Harness 差在哪。

1. [Token、Embedding、RAG、Agent、MCP](agents/五大核心概念.md)
2. [Agent 本质：Model 加 Harness](agents/本质.md)
3. [Model：推理、提示词和 Function Calling](agents/Model.md)
4. [动手：从 Function Calling 跑到 Research Agent](agents/动手版.md)
5. [实战：把各部件收成一个 Research Agent](agents/实战.md)
6. [同一个模型，换套马具](agents/同一个模型换套马具.md)
7. [自进化：改 Harness，还是改权重](agents/自进化.md)
8. [终态：一群有边界的工人，不是一个超级大脑](agents/终态.md)

终态放在最后。它引用后面各篇的闸，适合读完再回来对一遍。

## 2. Running

这一轮怎么走、怎么停、进程死了怎么续。

1. [图编排：循环要能停、能分支、能存档](running/图编排.md)
2. [Planning：这一轮先想完，还是边做边想](running/Planning.md)
3. [长时运行：跑了四十分钟，进程可以死](running/长时运行.md)
4. [级联故障：一个超时不要拖死整张图](running/级联故障.md)
5. [编排框架：先看执行模型，再看名字](running/编排框架.md)
6. [部署：有 Checkpoint 的服务不能当无状态 API 扩](running/部署.md)
7. [Runtime：模型之外，Agent 真正活着的那一层](running/全景.md)

## 3. Tools

1. [Tool Use：从一次调用到生产级循环](tools/ToolUse.md)

## 4. Handoffs

先决定要不要再开一个循环，再决定控制权、数据和跨进程协议。

1. [多 Agent：什么时候值得再开一个循环](handoffs/多Agent.md)
2. [Supervisor 还是 Swarm](handoffs/Supervisor.md)
3. [多 Agent 一致性：两个写者，一条数据](handoffs/多Agent一致性.md)
4. [通信：共享 State，还是传消息](handoffs/通信.md)
5. [A2A：跨框架的任务，不是再走一遍 MCP](handoffs/A2A.md)

## 5. Guardrails

1. [安全：一句话不该变成删库](guardrails/安全.md)
2. [权限：票跟着任务走，不跟着 Agent 的名字走](guardrails/权限.md)

## 6. Sessions

1. [Memory：对话、Checkpoint 和长期记忆](sessions/Memory.md)
2. [State：并行写同一个字段时谁赢](sessions/State设计.md)
3. [上下文工程：窗口里放什么](sessions/上下文工程.md)

## 7. Tracing

1. [可观测：生产上要能指出是哪一步](tracing/可观测.md)
2. [评估：输出每次不同，测什么才算数](tracing/评估.md)
3. [成本：五万的账单是四层叠出来的](tracing/成本.md)

## 8. MCP

1. [MCP：工具从编译期依赖变成运行时发现](mcp/MCP.md)

## 9. HITL

1. [HITL：三种介入，不是一个确认框](hitl/HITL.md)

## 10. Sandbox

1. [沙箱：进程、容器、虚机挡住的不是同一件事](sandbox/沙箱.md)

第一次读可以只走 Agents 的前四篇，再读 Running 的图编排。查某一个部件，进对应目录。进程死了、换了机器、人等了几天还要续上，读 [全景](running/全景.md)。
