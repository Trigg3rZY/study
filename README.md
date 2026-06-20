# study

个人工程学习笔记。

> 整套笔记大致分三条线:**① 判断与工程心智**(用 AI 造东西时,人该练什么)、**② AI / LLM 技术**(把 AI 装进产品的工程)、**③ Agentic 工作流**(用 AI 造东西的协作方式)。
> [Harness Engineering](2026-06-18-harness-engineering.md) 是贯穿三条线的总论:**模型是不确定的内核,工程的价值在它周围那一整圈确定性外壳。**

**给没经历过这些实践的人,建议阅读顺序**:先读总论(Harness Engineering)建立"确定性外壳"的框架 → 再读 ① 三篇(判断 / 决策 / 产品)建立心智 → 按需读 ②③ 的具体技术与协作。

---

## 总论

- [2026-06-18-harness-engineering.md](2026-06-18-harness-engineering.md) —— Harness Engineering 工程实践总结:模型是不确定的内核,工程的价值在它周围的那一整圈确定性外壳。涵盖「用 AI 造 app(agentic coding)」与「把 AI 装进 app(product harness)」两个方向的实战教训,并对标 Anthropic / OpenAI 的 evals 驱动范式。

## ① 判断与工程心智(用 AI 造东西时,人该练什么)

> 三篇是一个递进的三部曲:**为什么判断是瓶颈 → 单个决策怎么判 → 把判断落到产品流程**。

- [2026-06-20-taste-and-judgment.md](2026-06-20-taste-and-judgment.md) —— 判断力与品味:执行被 AI 商品化后,价值挤向"决定做什么 + 定义什么算好",这是唯一不被商品化的能力。讲清「建造之环」(taste / 执行 / verification / architecture)、reps(判断靠"亲自判+撞反馈"的次数累积,AI 会抄走 reps 让你变钝)、以及"多数人缺的不是 taste,是 taste 的延迟"。
- [2026-06-20-deciding-with-ai.md](2026-06-20-deciding-with-ai.md) —— 和 AI 协作的决策纪律:在"全盘接纳"与"全盘自己判"之间靠**分诊**——单向门(难回退/定义身份)自己拥有并主动跟 AI 吵一架,双向门(可逆/便宜)交给 AI 别浪费 rep。附"换一种问法(要反方/选项+trade-off/steelman)"的杠杆用法。
- [2026-06-20-product-judgment.md](2026-06-20-product-judgment.md) —— 从意图到产品:执行可外包、意图不可。PRD = 产品层的确定性外壳;non-goals 比 goals 杠杆高;锚定效应(身份 settle 前进场的特性会变成身份);两类"迟到的发现"(spec-可知 vs 上手才涌现);把 verification 抬到产品层 + 漂移 tripwire。含三轮 MVP 演化案例。

## ② AI / LLM 技术(把 AI 装进产品)

- [2026-06-19-llm-integration-and-model-selection.md](2026-06-19-llm-integration-and-model-selection.md) —— LLM 接入与「让用户自选模型」:模型只是可替换后端,工程在它外面那层抽象(统一接口、能力感知、BYO-key、结构化输出)。讲清 function calling / tool_choice / temperature / structured outputs / reasoning effort,对比 Vercel AI SDK / LiteLLM / OpenRouter,并给出 evidata 的 BYO-key 多模型落地路线。
- [2026-06-19-mcp-model-context-protocol.md](2026-06-19-mcp-model-context-protocol.md) —— MCP(Model Context Protocol):工具/数据接入的开放标准(≠ 模型接入,两者正交)。澄清 host/client/server 角色与「大模型在 host 侧、server 通常不碰模型」的关键认知,覆盖原语、传输(stdio + Streamable HTTP,SSE 已弃)、spec 修订史、安全与对 evidata 的意义。

## ③ Agentic 工作流(用 AI 造东西的协作方式)

- [2026-06-20-agentic-multi-agent-workflows.md](2026-06-20-agentic-multi-agent-workflows.md) —— Agentic 工作流与多 agent 并行:瓶颈不是「开几个 agent」而是协调+集成+评审(分解质量 > agent 数量)。讲清并行三难——去重(编排者发牌 / 认领租约)、依赖序(显式编码 + 串行合并队列)、通信(blackboard,靠产物不靠闲聊),推荐「lead + worktree workers + 串行合并」拓扑,并解释 GitHub Projects 当「免费 Jira」的用法。
