# Agent

除 Model 之外，Agent 落地要做的事：Harness、State、工具调用、多 Agent、安全与评测。面向后端开发者。

网站阅读版：[aqjszz.com/agent](https://aqjszz.com/agent/)。

## 目录职责

```
tutorial/       入门教程：分章精读 + 一篇动手版，从概念到第一个能跑的 Agent
architecture/   架构系列：每篇一个设计取舍，默认已读过教程
harness/        案例拆解：拿真实开源 Harness 对照架构系列
interview/      面试题：按面试问法组织，复习用
image/          配图：只被文章引用，不单独阅读
```

- **tutorial/** 讲 Agent 由什么组成、每块怎么写，有两种读法。分章精读：五大核心概念是入门铺垫，第 1–7 章依次讲本质、Model、Tool Use、Memory、Planning、多 Agent 和实战项目，重在为什么这样设计。动手版：同一条主线压成一篇，每一步都是能直接运行的代码，附环境搭建和概念速查，适合先跑通再回头补原理，或者读完分章对着敲一遍。
- **architecture/** 讲系统怎么设计。不重复教程里的 API 用法，每篇只盯一个取舍，给出判断依据。第 0 篇图编排是后面各篇的底座。
- **harness/** 放对真实项目的拆解。架构系列讲原则，这里看某个项目在这些原则上怎么取舍。
- **interview/** 和前面几块内容有交叉，但按面试问法组织，适合回头复习。
- **image/agent/** 是教程、架构系列和案例拆解共用的配图，**image/interview/** 是面试题配图。这些路径已被网站和外部文章引用，文件不要改名或挪动。

第一次读：tutorial → architecture → harness。准备面试：先看 interview，答不上来的再回对应章节。

## 教程

分章精读：

- [入门铺垫：Token、Embedding、RAG、Agent、MCP](tutorial/五大核心概念.md)
- [第1章 Agent 本质](tutorial/第1章_Agent本质.md)
- [第2章 Model 深度解析](tutorial/第2章_Model深度解析.md)
- [第3章 Tool Use](tutorial/第3章_Tool_Use.md)
- [第4章 Memory](tutorial/第4章_Memory.md)
- [第5章 Planning](tutorial/第5章_Planning.md)
- [第6章 Multi Agent](tutorial/第6章_Multi_Agent.md)
- [第7章 实战项目](tutorial/第7章_实战项目.md)

动手版：

- [一篇跑通 Agent：从 Function Calling 到 Research Agent，全程可运行代码](tutorial/动手版_一篇跑通Agent.md)

## 架构系列

完整目录和每篇的一句话说明见 [architecture/README.md](architecture/README.md)。

- [第 0 篇：LangGraph 不是 LangChain 的插件](architecture/第00篇_图编排.md)
- [第 1 篇：State 设计](architecture/第01篇_State设计.md)
- [第 2 篇：HITL](architecture/第02篇_HITL.md)
- [第 3 篇：上下文工程](architecture/第03篇_上下文工程.md)
- [第 4 篇：长时运行](architecture/第04篇_长时运行.md)
- [第 5 篇：多 Agent 一致性](architecture/第05篇_多Agent一致性.md)
- [第 6 篇：级联故障](architecture/第06篇_级联故障.md)
- [第 7 篇：评估](architecture/第07篇_评估.md)
- [第 8 篇：安全](architecture/第08篇_安全.md)
- [第 9 篇：Supervisor vs Swarm](architecture/第09篇_Supervisor.md)
- [第 10 篇：通信架构](architecture/第10篇_通信.md)
- [第 11 篇：权限模型](architecture/第11篇_权限.md)
- [第 12 篇：编排框架](architecture/第12篇_编排框架.md)
- [第 13 篇：可观测](architecture/第13篇_可观测.md)
- [第 14 篇：成本](architecture/第14篇_成本.md)
- [第 15 篇：部署](architecture/第15篇_部署.md)
- [第 16 篇：沙箱](architecture/第16篇_沙箱.md)
- [第 17 篇：MCP](architecture/第17篇_MCP.md)
- [第 18 篇：A2A](architecture/第18篇_A2A.md)
- [第 19 篇：自进化](architecture/第19篇_自进化.md)
- [第 20 篇：终态](architecture/第20篇_终态.md)

## 案例拆解

- [DeepSeek Harness：同一个模型，换套马具，Agent 能差一截](harness/同一个模型换套马具.md)

## 面试

- [Agent 面试题](interview/Agent面试题.md)
