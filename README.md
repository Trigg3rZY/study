# study

个人工程学习笔记。

> **主轴是"做事的三层":心智(为什么)→ 方法论(把心智变成可重复的"如果…就…")→ 工程实践(具体落地的手艺)。**
> [Harness Engineering](2026-06-18-harness-engineering.md) 是贯穿三层的总论(模型是不确定内核,工程的价值在它周围那圈确定性外壳);另有一组 LLM/MCP 技术参考。

**给没经历过这些实践的人,建议阅读顺序**:总论(确定性外壳)→ 心智三篇(建立判断)→ 方法论(把心智落地的桥)→ 工程实践(照着配)→ 按需读技术参考。

---

## 总论

- [2026-06-18-harness-engineering.md](2026-06-18-harness-engineering.md) —— Harness Engineering:模型是不确定的内核,工程的价值在它周围那一整圈确定性外壳。涵盖「用 AI 造 app(agentic coding)」与「把 AI 装进 app(product harness)」两个方向,对标 Anthropic / OpenAI 的 evals 驱动范式。

## 心智层(为什么 —— 该练什么、怎么判断)

> 三部曲递进:**为什么判断是瓶颈 → 单个决策怎么判 → 把判断落到产品**。

- [2026-06-20-taste-and-judgment.md](2026-06-20-taste-and-judgment.md) —— 判断力与品味:执行被 AI 商品化后,价值挤向"决定做什么 + 定义什么算好"。建造之环、reps(判断靠"亲自判+撞反馈"累积,AI 会抄走 reps)、"缺的不是 taste 是 taste 的延迟"。
- [2026-06-20-deciding-with-ai.md](2026-06-20-deciding-with-ai.md) —— 决策纪律:在"全盘接纳"与"全盘自己判"之间靠分诊——单向门自己拥有、双向门交给 AI;换问法(反方/选项+trade-off/steelman)把 AI 当弹药。
- [2026-06-20-product-judgment.md](2026-06-20-product-judgment.md) —— 从意图到产品:执行可外包、意图不可。PRD = 产品层确定性外壳;non-goals > goals;锚定效应;spec-可知 vs 上手才涌现;verification 抬到产品层 + 漂移 tripwire。含三轮 MVP 案例。

## 方法论层(桥 —— 把心智变成可重复的方法)

- [2026-06-20-mindset-to-methodology.md](2026-06-20-mindset-to-methodology.md) —— 从心智到方法论到工程实践:三层模型 + 一张"心智→方法论→实践"落地地图 + **自己造桥的元方法**(写成"如果X就Y"→ 找可观测触发器 → 编译进结构)。心智与实践之间的连接组织。

## 工程实践层(具体手艺 —— 照着配)

- [2026-06-20-delivery-harness.md](2026-06-20-delivery-harness.md) —— 交付外壳:把"确定性外壳"从产品延伸到开发流程本身。AI 写得快→错误注入也便宜→门控必须更硬更自动。分支/合并纪律、必需检查、CodeQL、评审闭环(thread-resolution)、供应链/密钥硬化、worktree 隔离、PR 作为契约。全部用实战配置打底,附落地清单。
- [2026-06-20-agentic-multi-agent-workflows.md](2026-06-20-agentic-multi-agent-workflows.md) —— Agentic 工作流与多 agent 并行:瓶颈是协调+集成+评审(分解质量 > agent 数量)。并行三难(去重/依赖序/通信)、orchestrator vs workflow、「lead + worktree + 串行合并」拓扑、GitHub Projects 当免费 Jira。

## 技术参考(把 AI 装进产品的具体技术)

- [2026-06-19-llm-integration-and-model-selection.md](2026-06-19-llm-integration-and-model-selection.md) —— LLM 接入与「让用户自选模型」:模型只是可替换后端,工程在外面那层抽象(统一接口、能力感知、BYO-key、结构化输出)。function calling / tool_choice / temperature / structured outputs / reasoning effort,对比 Vercel AI SDK / LiteLLM / OpenRouter。
- [2026-06-19-mcp-model-context-protocol.md](2026-06-19-mcp-model-context-protocol.md) —— MCP:工具/数据接入的开放标准(≠ 模型接入,两者正交)。host/client/server 角色、原语、传输(stdio + Streamable HTTP)、安全。
