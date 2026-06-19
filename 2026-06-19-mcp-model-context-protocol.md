# MCP(Model Context Protocol)学习总结

> 副标题:MCP 是"工具/数据接入"的开放标准,不是"模型接入"——它把 M 个应用 × N 个工具的定制集成,变成 M+N。最关键的认知:**大模型在 host/client 那一侧,MCP server 通常不碰模型。**
>
> 日期:2026-06-19 ｜ 来源:evidata 设计讨论中对 MCP 的一次澄清(纠正了"server 调大模型"这个常见误解)
>
> 配套阅读:同目录的 [2026-06-19-llm-integration-and-model-selection.md](2026-06-19-llm-integration-and-model-selection.md)(模型层;和本篇的工具层**正交**)。

---

## 0. 一句话定位

> **MCP = 把"AI 应用 ↔ 外部工具/数据/工作流"的连接标准化的开放协议。** 官方比喻:**"AI 应用的 USB-C 口"**——就像 USB-C 给设备一个统一接口,MCP 给 AI 应用一个统一方式去连数据源、工具、工作流。
>
> 它**不管你用哪个大模型**(那是模型层的事),只管"工具和上下文怎么被发现、被暴露、被调用"。底层是 **JSON-RPC 2.0**。

---

## 1. 它解决什么问题

- 在 MCP 之前:**每个 AI 应用 × 每个工具/数据源都要写一套定制集成** —— 集成数随组合爆炸,难以规模化。
- MCP 用**一个标准协议**取代这些碎片化集成 → 官方话术:**"replacing fragmented integrations with a single protocol" / "build once and integrate everywhere"**。
- 工具方实现一次 **MCP server**;应用方做 **MCP client**;**任意 client 能用任意 server**。

> ⚠️ 常见的 **"M×N → M+N"** 是**社区通俗说法,不是官方措辞**(官方用"碎片化集成→单一协议")。意思对,但别当成官方原话。

---

## 2. 架构与角色:**大模型在哪一侧?**(最容易搞反)

官方架构是 **client-host-server**(JSON-RPC 2.0):

| 角色 | 职责(据官方 spec) | 有没有大模型 |
|---|---|---|
| **Host(宿主)** | "协调 AI/LLM 集成与 sampling"、跨 client 聚合上下文 | **✅ 大模型在这侧** |
| **Client(客户端)** | host 内的连接器,**与某个 server 1:1**,转发消息 | 无(代表 host 与 server 通信) |
| **Server(服务端)** | 暴露 **resources / tools / prompts**;需要推理时"**通过 client 请求 sampling**" | **❌ 通常不碰大模型** |

```
你 ⇄ Host(Claude Desktop / Cursor / Claude Code)
        ├── 直连大模型(host 持有 LLM)
        └── MCP client ──JSON-RPC──▶ MCP server(文件/GitHub/DB…只给工具与数据,无 LLM)
```

> **"MCP server 调用/内含大模型" = 错。** host 协调 LLM;server 只是工具/数据提供方。还有一条 spec 明确的隔离:**server 看不到完整对话,也看不到别的 server;完整对话历史留在 host。**

### 唯一的例外:Sampling(采样)
spec 里 server **可以**反过来请求推理,但方式是:**`sampling/createMessage` 发给 client,由 client 转发给 LLM**。关键点(官方原文要点):
- "**无需 server 持有 API key**"——模型仍在 client/host 侧;
- "**SHOULD 始终有 human-in-the-loop**,能拒绝 sampling 请求"。
- 设计意图原文:让 server 作者"想用语言模型、但保持 **model-independent**,不必在 server 里塞一个 LLM SDK"。

→ 所以即便 server "触发了推理",**模型依然在 client 侧**,这正是"server 不碰模型"规则的精确边界。

---

## 3. 原语(Primitives)

### Server 暴露(三个)
| 原语 | 控制方 | 定义(官方要点) |
|---|---|---|
| **Tools** | 模型控制 | 可被语言模型调用的函数:查库、调 API、做计算 |
| **Resources** | 应用驱动 | 给模型当上下文的数据:文件、库 schema、应用信息 |
| **Prompts** | 用户控制 | 结构化的消息/指令模板 |

### Client/Host 暴露(三个)
| 原语 | 定义(官方要点) |
|---|---|
| **Sampling** | server 经 client 请求 LLM 补全(见 §2;无需 server key、HITL) |
| **Roots** | client 把文件系统"根"暴露给 server,**界定 server 能操作的边界** |
| **Elicitation** | server 在交互中**经 client 向用户索取额外信息**;**MUST NOT 用它要敏感信息**。`2025-06-18` 引入,`2025-11-25` 加了 URL 模式 |

---

## 4. 传输(Transports)——一个最容易过时的点

- **stdio(本地)**:client 把 server 当子进程拉起,走 stdin/stdout 收发 JSON-RPC。"clients SHOULD 尽量支持 stdio"。
- **Streamable HTTP(远程)**:**单个 MCP 端点**支持 POST + GET,SSE 可选用于流式。

> ⚠️ **过时警告**:最初(2024-11-05)的 HTTP 传输叫 **"HTTP with SSE"**(两个端点)。**`2025-03-26` 修订用 "Streamable HTTP" 取代了它**(spec 原文:"This replaces the HTTP+SSE transport from protocol version 2024-11-05")。**现在的两种传输是 stdio + Streamable HTTP,别再说 "MCP 用 SSE"。**

### Spec 修订史(把日期搞对)
| 修订 | 状态(2026-06) | 要点 |
|---|---|---|
| 2024-11-05 | 已被取代 | 初版;stdio + HTTP+SSE |
| 2025-03-26 | 已被取代 | **Streamable HTTP**(取代 SSE);OAuth 2.1;JSON-RPC 批处理;工具注解 |
| 2025-06-18 | 已被取代 | **Elicitation**;结构化工具输出;OAuth Resource Server + RFC 8707;**移除**批处理;安全最佳实践页 |
| **2025-11-25** | **当前最新(正式)** | Tasks(实验);Extensions;URL 模式 elicitation;sampling-with-tools |
| 2026-07-28 | **候选(尚未正式)** | 无状态协议核心;破坏性变更 |

> 趣点:JSON-RPC 批处理在 2025-03-26 **加入**、2025-06-18 又**移除**。
> 另:**"最新是 2025-06-18" 已过时**——当前正式版是 **2025-11-25**。

---

## 5. 来历与采纳(澄清"是不是 Claude 专属")

- **Anthropic 创建并开源,2024-11-25 发布**(作者 David Soria Parra、Justin Spahr-Summers)。
- **不是 Claude 专属**:OpenAI(Agents SDK,2025-03-26)、Google DeepMind(2025-04-09 表态支持)、Microsoft(Copilot Studio 2025-03-19、C# SDK、Windows 11 预览)、IDE(VS Code GA 2025-07-14;JetBrains/Eclipse/Xcode GA 2025-08-13)陆续接入。
- **已捐给 Linux Foundation**(2025-12-09,并入 Agentic AI Foundation)——现在是 "Model Context Protocol a Series of LF Projects, LLC",**厂商中立**。
- **官方 SDK**:TypeScript `@modelcontextprotocol/sdk`(写本文时稳定版 1.29.0),另有 Python/Java/Kotlin/C#/Go/Ruby/Rust/Swift/PHP 等共十种。"实现 MCP" = 用这些 SDK,而不是手撸协议。

---

## 6. MCP vs function calling:**正交**

- MCP 标准化的是"**工具/资源/prompt 怎么被发现和暴露**"(JSON-RPC 方法:`*/list` 发现、`*/get` 取、`tools/call` 执行)。
- **最强的一句官方话**:"MCP focuses solely on the protocol for context exchange—**it does not dictate how AI applications use LLMs** or manage the provided context."
- 它**不绑定**:(a) 你用哪个 LLM;(b) 某家厂商的 function-calling schema。**host 负责把 MCP 的 `tools/list` 结果,映射成所选模型的原生工具调用格式。**
- 一句话:**MCP 是"传输 + 发现"标准(JSON-RPC over stdio / Streamable HTTP);function calling 是"模型怎么表达调用"。MCP 在它外面/下面,不替代它。**

---

## 7. 安全(spec 明确警告的)

MCP 自己**无法在协议层强制**这些,但要求实现方做到:
- **用户同意 + 工具权限**:"Hosts must obtain explicit user consent before invoking any tool";工具=任意代码执行,需谨慎。
- **prompt injection / 不可信描述**:工具描述/注解"除非来自可信 server,否则应视为不可信"。
- **远程 server 授权**:OAuth 2.1 框架;server 作为 **OAuth Resource Server** + **RFC 8707 Resource Indicators**(防恶意 server 套取 token)。
- **confused deputy / token passthrough**:代理 server 必须 per-client 同意;**禁止 token 透传**(不得接受不是发给自己的 token)。
- **会话**:必须校验所有入站请求;**不得用 session 做认证**;session id 要安全且非确定。
- 还警告:SSRF、本地 server 一键安装的供应链风险、最小权限。

> 对 evidata 尤其相关:**若把 evidata 暴露成 MCP server,等于多开了一个查询入口——上面这些(同意、授权、最小权限、注入)必须照样套上,且我们自己的 SafetyGate/只读/脱敏一个都不能少。**

---

## 8. 和 evidata 的关系(现在不需要,以后可选)

**现状:evidata 完全没用 MCP**——agent 调工具是**进程内**实现的,不是 MCP。概念上的对应(仅类比):

| MCP 角色 | evidata 里≈谁 |
|---|---|
| Host / Client(持模型、编排) | `apps/web` → `InvestigationService` → **`AgentRunner`** |
| "模型"那一侧(非 MCP 角色) | `AgentProvider`(`@evidata/provider-openai`) |
| Server 能力(但**内嵌**,非跨进程) | `Connector`/`QueryExecutor` + `SafetyGate`/`Redactor` |

**何时才值得上 MCP**(= 跨"别人做的应用"边界时,两个方向):
- **evidata 当 server**(产品卖点):把"受治理的只读查询"暴露成 MCP server → Claude Desktop / Cursor 等**透过我们的安全/治理层**查数据。可暴露低层 `run_sql`(我们不调模型)或高层 `ask_data`(内部跑我们自己的 agent,这时才用我们配的模型)。
- **evidata 当 client**:让我们的 agent 挂上**外部 MCP server** 获得更多工具。

> 重要边界认知:**自用不需要 MCP**(进程内函数调用更简单);**只有要和外部 AI 生态即插即用时,MCP 的标准化才划算**。它**不**解决"让用户选模型"——那是另一篇的事。

---

## 9. 推荐读物(已核对｜截至 2026-06-19)

> 经一次 deep-research(fan-out → 抓官方 spec → 对抗式核验);**spec 修订是日期戳的,注意别引用过时版本**。

- 入门/定义:https://modelcontextprotocol.io/docs/getting-started/intro
- 架构:https://modelcontextprotocol.io/specification/2025-11-25/architecture ｜ 学习版 https://modelcontextprotocol.io/docs/learn/architecture
- 当前修订/版本:https://modelcontextprotocol.io/specification/versioning
- 传输(SSE→Streamable HTTP 的更替):https://modelcontextprotocol.io/specification/2025-03-26/basic/transports
- 原语:`.../2025-06-18/server/{tools,resources,prompts}` ｜ `.../client/{sampling,roots,elicitation}`
- 安全:`.../basic/security_best_practices`
- Anthropic 公告(2024-11-25):https://www.anthropic.com/news/model-context-protocol
- 治理(已入 Linux Foundation):https://modelcontextprotocol.io/community/governance
- TS SDK:https://www.npmjs.com/package/@modelcontextprotocol/sdk ｜ 各语言:https://github.com/orgs/modelcontextprotocol/repositories

### 最容易踩的 5 个误解 / 过时事实
1. **"MCP 用 HTTP+SSE"** —— 过时;2025-03-26 起被 **Streamable HTTP** 取代,现在是 stdio + Streamable HTTP。
2. **"MCP server 调用/内含大模型"** —— 错;模型在 host/client 侧,server 只经 **sampling** 请求(无 server key、HITL)。
3. **"最新 spec 是 2025-06-18"** —— 过时;当前正式版 **2025-11-25**(2026-07-28 为候选)。
4. **"MCP 是 Claude / Anthropic 专属"** —— 错;OpenAI/Google/Microsoft/各 IDE 都接了,已是 **Linux Foundation** 项目,且**显式 model-independent**。
5. **"M×N → M+N 是官方说法"** —— 是社区通俗版;官方是"碎片化集成→单一协议"。

---

## 10. 一句话带走

> **模型层的问题(选哪个 LLM、BYO-key、effort)用模型抽象(Vercel AI SDK)解决;工具/数据层的互通(让 evidata 被别的 AI 应用使用、或消费外部工具)才用 MCP。两者正交。** 而无论哪层,evidata 的安全核心("AI 提议、应用裁决")都不变——这是它作为"受治理数据源"的立身之本。
