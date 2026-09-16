# 第 17 篇：MCP，工具从编译时依赖变成运行时发现

---

第 8 篇提过 MCP SDK 的 STDIO 漏洞，协议方说「预期行为，开发者自己过滤」。这一篇不审那个 CVE，审 **MCP 改了工具的架构位置。**

以前工具是你代码里 import 的函数，发版时钉死。MCP 把工具变成运行时连上的服务器：模型问「你有什么」，服务器用 JSON-RPC 列清单，再 `tools/call`。发现和调用都在线上发生。灵活，也把第 11 篇的权限和第 16 篇的沙箱边界往后推了一截——推到你不一定控制的进程上。

官方规格在 modelcontextprotocol.io。传输常见 STDIO（本地子进程）和 Streamable HTTP（远端）。不管哪种，对 Agent 图来说 MCP 服务器就是一个 **动态工具表**。

---

### 一、动态工具表带来什么

**好处：** 换检索后端不用改图、不用发版。团队按服务边界拆 MCP 服务器，Agent 只持有入口。

**代价：**

1. 工具名和描述进 prompt，间接注入可以藏在 **工具描述** 里，不只藏在网页。第 8 篇的外部内容，现在包括 MCP list 的返回。
2. 权限：服务器列出 `delete_repo`，主体授权若按「服务器」而不是「action」，等于把删库能力发现出来就接上。
3. 供应链：谁有权被你的宿主连接。ClawHub 上的恶意技能是同一类问题的市场版。
4. STDIO 把「命令行参数」当传输配置，配置即代码。第 8 篇的 spawn 没有校验，不是实现疏忽，是把配置通道设计成了 shell。

---

### 二、图里怎么接才不炸

把 MCP 当运行时插件，不要当「模型直接看见的全世界」。

```python
ALLOWED_SERVERS = {"search", "tickets"}  # 显式名单，不是扫描本地目录
ALLOWED_TOOLS = {
    "search": {"web_search"},
    "tickets": {"ticket.get", "ticket.comment"},
}

def mcp_tools(server: str) -> list:
    raw = mcp_list(server)  # JSON-RPC tools/list
    return [t for t in raw if t["name"] in ALLOWED_TOOLS[server]]
```

list 的结果进 State 前当 untrusted：描述字段做长度截断和注入打分。调用时仍走第 11 篇的票：`ticket.comment` 要有对应 action，资源是 `ticket:id`。MCP 不发票，只是通道。

STDIO 服务器的二进制路径来自你的配置仓库，不来自会话、不来自网页、不来自模型输出。HTTP 服务器要 TLS、要身份，不要局域网扫端口「发现」。

---

### 三、和本系列的接口

- 第 3 篇：工具描述占窗口。MCP 一列 50 个工具，窗口先被说明书占满。要按任务票过滤后再塞 prompt。
- 第 6 篇：远端 MCP 超时、限流，按服务器熔断，不要按「所有工具」一把熔。
- 第 13 篇：span 记 `mcp.server`、`mcp.tool`、协议错误码。
- 第 16 篇：STDIO 服务器进程本身就该在沙箱里，不要跟宿主同 uid。

---

### 收尾

MCP 把工具发现从编译期挪到运行期。发现必须白名单，描述必须当不可信输入，调用必须另验票。协议不管这三件事，图必须管。

下一篇跨过工具，到 Agent 之间：A2A 让不同框架的 Agent 按协议说话，而不是共用一个 Python 进程。

我是Q，下篇见。
