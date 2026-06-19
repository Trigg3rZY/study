# LLM 接入与「让用户自选模型」工程总结

> 副标题：模型只是一个可替换的后端——真正的工程在它**外面那层抽象**：统一接口、能力感知、BYO-key、结构化输出。把"用哪个模型"做成配置,而不是写死。
>
> 日期：2026-06-19 ｜ 来源：自托管 AI 数据门户 **evidata** 的一次真实设计讨论(从"为什么钉死 deepseek-chat"一路追到"如何让用户自带 key、自选模型与 effort")
>
> 配套阅读：同目录的 [2026-06-19-mcp-model-context-protocol.md](2026-06-19-mcp-model-context-protocol.md)(MCP 是**工具/数据层**,和本篇的**模型层**正交,别混)。

---

## 0. 一句话定位

> **"接入 LLM" 不是"调一个 API",而是"在一个不稳定、各家各异、还会过期的后端前面,造一层稳定抽象"。** 这层抽象要解决四件事:①统一不同厂商的接口;②感知每个模型支持什么(工具/结构化/推理);③安全地管住用户自带的 key;④把"用哪个模型"变成**运行时配置**而非编译期常量。

evidata 的教训:一开始我们手写了一个 OpenAI 兼容客户端、钉死 `deepseek-chat`、还打了一堆某家模型的兼容补丁(JSON 修复、多 tool_call 去重)。**对单一 provider 没问题,但一旦要"对接任意模型",这条路就不 scale 了。** 业界的答案是:**别自己造网关,用成熟的抽象层。**

---

## 1. 分清两个层(最容易混的地方)

| 层 | 管什么 | 方案 | 关键词 |
|---|---|---|---|
| **模型层(本篇)** | 用**哪个 LLM**、谁的 key、多大 effort | 统一 SDK / 网关 | provider、tool_choice、structured output、reasoning effort |
| **工具/数据层** | agent 怎么**拿到工具和上下文** | MCP(可选) | host/client/server、resources、tools |

> 这俩**正交**。"让用户选模型"是模型层的事;MCP 一点都帮不上。详见 MCP 那篇。

---

## 2. 三个最常被问的名词(大白话)

### 2.1 function calling / tools / `tool_choice`
- **function calling(工具/函数调用)**:不让模型吐自由文本,而是给它一份"工具清单"(带 JSON schema 的函数,如 `run_sql(query)`)。模型回的是一个**结构化调用**`{name, arguments}`;你的代码执行、把结果喂回、继续循环。好处:拿到**机器可解析、可校验**的东西,而不是从散文里正则抠。
- **`tool_choice`**(OpenAI Chat Completions 的取值,已核对):
  - `"none"` —— 不调工具(无 tools 时的默认);
  - `"auto"` —— 模型自己决定调不调(有 tools 时的默认);
  - `"required"` —— **必须**调某个工具;
  - `{"type":"function","function":{"name":"X"}}` —— 强制点名调 X。
- ⚠️ **形状坑**:OpenAI 新文档围绕 **Responses API** 写,强制工具是 `{"type":"function","name":"X"}`(无嵌套);**Chat Completions 仍是嵌套** `{"type":"function","function":{"name":"X"}}`。别混。
- ⚠️ `parallel_tool_calls` 默认 `true`;想"一轮最多一个工具"要设 `false`(o 系列、GPT-5 `minimal` effort 下不支持)。

### 2.2 temperature(温度)
- 范围 **0–2**,默认 **1**;控制随机性。`0` ≈ 聚焦/确定(但**不保证逐字节可复现**,后端有非确定性)。
- 官方建议:**改 temperature 或 top_p,别两个一起改**。
- ⚠️ **推理模型(o 系列、GPT-5 推理款)直接拒绝 `temperature`/`top_p`**(固定 1.0),只认 `reasoning_effort` + `max_completion_tokens`。

### 2.3 structured outputs(结构化输出)
- **JSON mode** `response_format:{type:"json_object"}` —— 保证是**合法 JSON**,但**不保证符合你的 schema**(还得在 prompt 里要求出 JSON,否则可能狂吐空白)。
- **Structured Outputs** `response_format:{type:"json_schema", json_schema:{… strict:true}}` —— **保证符合你给的 JSON Schema**(底层是**受限解码 constrained decoding**)。strict 模式有子集约束(每个 object 要 `additionalProperties:false`、所有字段都 `required`、可选用 `null` 联合表达)。
- 同样能用在**工具参数**上(tool 定义里 `strict:true`)。
- **关键价值**:对 function-calling 不靠谱的模型,用结构化输出能**绕过"工具调用可靠性"**,直接拿到保证形状的对象。这正是"能力感知 A/B 策略"里的 B 路(见 §6)。

---

## 3. 推理模型 + "effort":**不是一个统一参数**(重要)

你要的"让用户选 effort",最大的坑是:**各家的"想得更久"控制结构完全不同**,且 2025–2026 越分越开。

| 厂商 | 机制 | 类型 | 取值(核对于 2026-06) |
|---|---|---|---|
| **OpenAI** | `reasoning_effort`(Chat)/`reasoning.effort`(Responses) | **字符串枚举** | `none/minimal/low/medium/high/xhigh`(**随模型而定的子集**;`minimal` 仅 GPT-5 初代;o 系列只有 low/med/high) |
| **Anthropic(经典)** | `thinking:{type:"enabled", budget_tokens:N}` | **整数 token 预算** | `≥1024 且 < max_tokens` |
| **Anthropic(最新款)** | `thinking:{type:"adaptive"}` + `output_config.effort` | **字符串枚举** | `low/medium/high(默认)/xhigh/max`;**新款拒绝 `budget_tokens`(400)** |
| **DeepSeek** | **换模型名**(经典无 effort 参数) | **model id** | `deepseek-reasoner`(思考)vs `deepseek-chat`(非思考) |

> **结论**:抽象层里存一个**抽象的 `effort`**,由每家 adapter 翻译(OpenAI 枚举 / Anthropic 整数预算或新枚举 / DeepSeek 换模型)。**绝不要假设有一个通用 `reasoning_effort`。**

### evidata 的一条诚实更正
我们之前的结论是"`deepseek-v4-flash`(思考款)拒绝 `tool_choice`,所以只能用 `deepseek-chat`"。**这条是时间敏感的,已部分过时**:据 DeepSeek 官方 `thinking_mode` 文档,**`deepseek-reasoner` 自 2025-05-28(R1-0528)起已支持 function calling / tool use**(早期 R1 确实不支持,旧的 `reasoning_model` 文档至今仍错写"Not Supported")。但仍有坑:
- 思考模式**仍不支持 `temperature`/`top_p`**(我们的循环要 `temperature:0`);
- 它把推理放在单独的 `reasoning_content` 字段,**tool-calling 跟进时必须把 `reasoning_content` 回传**,否则 400;
- "支持 `tools`(auto)" 和 "支持强制 `tool_choice:'required'`" 不一定等价——我们的循环用的是 `required`,这点要**重新实测**;
- `deepseek-chat`/`deepseek-reasoner` 这两个别名现在路由到 `deepseek-v4-flash`,且**计划 2026-07-24 退役**。

→ 教训:**"某模型不支持 X" 这类结论要标注日期并定期复验**;生态变得很快。这也正是为什么要把模型做成**可配置 + 能力感知**,而不是把某个结论焊死在代码里。

---

## 4. OpenAI 兼容 API = 事实标准(先吃透这条)

最省事的多模型路线:**保留 OpenAI 客户端,只换 `base_url` + `api_key` + 模型名。** 已核对的兼容端点:

| 厂商 | base_url | 备注 |
|---|---|---|
| OpenAI(基准) | `https://api.openai.com/v1` | — |
| DeepSeek | `https://api.deepseek.com`(也 `/v1`) | 明确"compatible with OpenAI" |
| Together AI | `https://api.together.ai/v1` | 模型 id 带命名空间,如 `openai/gpt-oss-20b` |
| Fireworks | `https://api.fireworks.ai/inference/v1` | 用 OpenAI 客户端 |
| Groq | `https://api.groq.com/openai/v1` | "mostly compatible"(缺 `logprobs`/`N>1` 等) |
| **Azure OpenAI** | `…/openai/v1` | **历史上不是纯 drop-in**(deployment 名 + api_version + Entra/key);新版 v1 GA API 正在拉平,**版本相关** |
| Ollama(本地) | `http://localhost:11434/v1/` | 要 key 但忽略 |
| vLLM(自托管) | `http://localhost:8000/v1` | "drop-in replacement" |
| LM Studio(本地) | `http://localhost:1234/v1` | 复用 OpenAI 客户端 |

> 所以"支持 OpenAI 兼容 + 可配 base_url/model"**一招就覆盖绝大多数模型**。evidata 现在就走这条(env 配 `OPENAI_BASE_URL`/`AGENT_MODEL`),只是还缺"多模型注册表 + UI 切换"。

---

## 5. 抽象层选型(别自己造网关)

| 方案 | 是什么 | 许可 / 形态 | 何时用 |
|---|---|---|---|
| **Vercel AI SDK**(首选,TS 栈) | TS 原生**库**,统一 provider + 工具 + 结构化 + 流式 + reasoning | **Apache-2.0**,自托管库,**不需要 Vercel 账号** | Next.js/TS 应用要"薄统一模型层" |
| LiteLLM | Python **SDK + 自托管 Proxy 网关**(虚拟 key/预算/日志) | MIT 核心 + 企业版 carve-out | 想要一个**跨语言的网关服务** |
| OpenRouter | **托管聚合网关**(一个端点/key,400+ 模型),支持 BYO-key(5% 费,每月前 100 万次免) | SaaS | 想要托管路由 + 回退,且能接受三方代理 |
| LangChain.js / LlamaIndex.TS | 编排框架 / 数据(RAG)框架 | OSS 库 | 需要多步 agent 编排 / RAG,而非单纯调模型 |

### 为什么对 evidata 选 Vercel AI SDK
- **OSS(Apache-2.0)、自托管库、无锁定**:它跑在我们自己服务器里,**用户的 key 服务器直连厂商**,不经第三方。
- **把各家差异都归一化了**:工具调用(`toolChoice`: `auto`/`required`/`none`/具名)、**结构化输出**、流式、**reasoning**——正好解决 §6 的 A/B 策略。
- **effort 仍是 per-provider**:通过 `providerOptions`(OpenAI `reasoningEffort`、Anthropic `thinking.budgetTokens`),不是统一字段——和 §3 一致。
- 放进我们现有的 `AgentProvider` port 后面,**runner / SafetyGate / 校验全不动**,还能删掉手写的那堆兼容补丁。
- ⚠️ **v6 的一个坑**:传**裸模型字符串**(如 `'anthropic/claude-...'`)默认会走 Vercel **AI Gateway**(那是个独立托管产品);**直接 import provider**(`anthropic('…')`)才完全绕过。自托管要走后者。

---

## 6. 能力感知的 A/B 执行策略(让"任意模型"真正可用)

我们的 agent 是**强制工具调用**循环(每轮 `tool_choice:'required'` 或点名 `final_answer`)。要支持"任意模型 + effort",runner 需要**两套策略**(都藏在 `AgentProvider` port 后面,按模型**能力标记**切换):

- **A 路 · 强制工具调用**(现状):模型支持 `tool_choice` → 走这条,每轮强制出结构。
- **B 路 · 结构化输出兜底**:模型不支持强制工具(部分推理款/小模型)→ 改用 **JSON schema 模式 + 校验 + 失败重试** 拿结构化输出。

> 这就是业界的 **capability-aware routing**:给每个模型打"支不支持 function calling / 结构化 / 视觉 / 上下文多大"的标记,据此路由或优雅降级。**安全不受影响**——不管哪条路、哪个模型,SQL 都得过同一道 SafetyGate(模型只提议、程序裁决)。

---

## 7. BYO-key:用户自带 key 的工程要点

业界对"自带 key"的标准做法(OWASP 对齐):
1. **静态加密**:AES-256-GCM(认证加密);最好**信封加密 + KMS/Vault**(DEK 加密数据、KEK 加密 DEK,集中轮换)。
   - evidata 已有 **CredentialVault**(M1 做的 AES-256-GCM,本来加密数据库凭证)——**正好复用**来加密每个模型的 key。
2. **绝不记日志**:key/token/连接串一律不进日志(脱敏/哈希/加密)。
3. **key 服务器侧直连厂商**:绝不放进浏览器/客户端代码;服务器在用时取出、直发 provider。
4. **配置形态**(对标 Continue/LibreChat/Open WebUI/Cursor):一组模型条目,每条带 `{ 名称, provider/baseURL, key引用, model, 参数, effort, 能力 }`;前端一个下拉切换。LibreChat 甚至有 `"user_provided"` 这种"运行时由用户填 key"的值——很贴合多租户/多用户 BYO-key。

---

## 8. evidata 的落地结论(对应 issue #106)

保留 `AgentProvider` port,加四样:
1. **多模型注册表**(形态≈现有 Connections):多条 `{名称, baseURL, key(用 Vault 加密), model, 参数, effort, 能力}`。
2. **采用 Vercel AI SDK** 塞到 port 后面,替掉手写客户端 + 兼容补丁。
3. **能力感知 A/B 策略** + **effort 归一化**(抽象 effort → per-provider 翻译)。
4. **模型 picker UI**(默认 + per 对话/数据源覆盖)+ provider 健康状态。

> 安全不变量:无论用户选哪个模型,**"AI 提议、应用裁决(校验/只读执行/脱敏/记证据)"** 这条线不动。所以"开放任意模型"**不削弱可信度**——这正是这套架构的价值。

---

## 9. 推荐读物(已核对｜截至 2026-06-19)

> 本节经一次 deep-research(fan-out 搜索 → 抓官方源 → 逐条对抗式核验)。**生态变化快,facts 标注了日期;凡"某模型不支持 X"务必复验。** 优先官方源。

- **Vercel AI SDK**:https://ai-sdk.dev/docs/introduction ｜ providers:https://ai-sdk.dev/docs/foundations/providers-and-models ｜ OpenAI 兼容 provider:https://ai-sdk.dev/providers/openai-compatible-providers ｜ 许可(Apache-2.0):https://github.com/vercel/ai/blob/main/LICENSE
- **OpenAI**:function calling https://developers.openai.com/api/docs/guides/function-calling ｜ structured outputs https://developers.openai.com/api/docs/guides/structured-outputs ｜ reasoning https://developers.openai.com/api/docs/guides/reasoning
- **Anthropic**:extended thinking https://platform.claude.com/docs/en/docs/build-with-claude/extended-thinking
- **DeepSeek**:thinking mode(权威,且更正了旧 reasoning_model 页)https://api-docs.deepseek.com/guides/thinking_mode
- **LiteLLM**:https://docs.litellm.ai/docs/ ｜ **OpenRouter** BYOK:https://openrouter.ai/docs/guides/overview/auth/byok
- **多模型配置范式**:Continue https://docs.continue.dev/reference ｜ LibreChat custom endpoints https://www.librechat.ai/docs/quick_start/custom_endpoints ｜ Open WebUI env https://docs.openwebui.com/reference/env-configuration/
- **安全**:OWASP Cryptographic Storage / Secrets Management / Logging cheat sheets(cheatsheetseries.owasp.org)

### 最容易踩的 5 个"用错术语/用过时 API"
1. **"`reasoning_effort` 是通用参数"** —— 错。OpenAI 枚举 / Anthropic 整数预算(新款又换枚举且拒绝预算)/ DeepSeek 换模型名。每家不同。
2. **混用 Responses 与 Chat Completions 的工具形状** —— 强制工具一个无嵌套、一个嵌套,复制粘贴就坏。
3. **"给推理模型设 temperature=0 求确定性"** —— 错,推理模型直接拒绝 temperature。
4. **"Vercel AI SDK 是 Vercel 托管服务 / 要账号 / 是 MIT"** —— 三条全错(Apache-2.0、自托管库、无需账号;AI Gateway 才是托管产品,且 v6 裸字符串模型默认走它)。
5. **"DeepSeek 推理款不支持 function calling"** —— **已过时**(2025-05 起支持;官方旧页仍错写)。但思考模式仍不支持 temperature、需回传 reasoning_content;`tool_choice:'required'` 是否支持要单独实测。

---

## 10. 诚实反思

### ✅ 想清楚的
- 早早把 provider 抽象成 port(`AgentProvider`),让"换模型层"成了局部替换、不动安全核心。
- 认清"安全是模型无关的"——开放任意模型不降低可信度。

### ⚠️ 能更好的
- 手写单 provider 客户端 + 钉死模型,是"对一个 provider 过拟合";应更早走 **OpenAI 兼容 + 标准 SDK**。
- "某模型不支持 X" 的结论没标日期、没定期复验,导致一个**可能已过时**的限制被当成长期事实(deepseek-reasoner 的 function calling)。**经验:凡涉及模型能力的判断,写下日期 + 复验机制。**
- 把"模型选择"做成 env 单值,而非配置式多模型——对 BYO-key 产品是早晚要补的债(已立项 #106)。
