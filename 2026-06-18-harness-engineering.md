# Harness Engineering 工程实践总结

> 副标题：模型是不确定的内核，工程的价值在它周围的那一整圈确定性外壳——从「用 AI 造 app」到「把 AI 装进 app」的同构实践。
>
> 日期：2026-06-18 ｜ 来源：用编码 agent（Claude Code）从零自托管 AI 数据门户 **evidata** 的一次真实工程实践

---

## 0. 一句话定义：什么是 harness engineering

> **模型是一个强大、但不确定、不可靠的组件；harness（马具/脚手架）= 你围绕它造的那一整圈确定性工程外壳。价值不在那一次模型调用，而在它周围的一切。**

这圈外壳至少包括六件事：

| 外壳组件 | 它在对抗什么 |
|---|---|
| **循环控制**（agent loop / step budget / force-finalize） | 模型不知道何时该停、何时该交付 |
| **工具与权限**（tool schema / 安全门 / 唯一执行路径） | 模型会"想"做危险的事，但不该"能"做 |
| **上下文组装**（context engineering） | 模型只看得到你喂给它的东西，喂错就答错 |
| **输出校验**（契约 / 断言 / 防御式解析） | 模型输出是概率串，不是可信结构 |
| **失败恢复**（修复 / re-prompt-once / 重试 / 兜底） | 模型会以各种方式坏掉，必须能优雅降级 |
| **评测与可观测**（evals / 埋点 / 计量） | 不能量化就不能改进，不能观测就不能定位 |

**两个方向只是这层外壳"包"谁，原则同构：**

- **方向一：用 AI 造 app（agentic coding）** —— harness 包住的是"写代码的 agent"，外壳是 spec / CI / PR / review / 诊断流程。
- **方向二：把 AI 装进 app（product harness）** —— harness 包住的是"产品里的 agent"，外壳是意图层 / 决策循环 / 安全门 / 契约校验 / 重试。

同一句话两种用法：**别想着让模型"自己搞定一切"，而是把模型放进一个它跑不出去、跑偏了能被发现、跑坏了能恢复的盒子里。**

---

## 1. 两方向共享的 5 条底层原则

无论外壳包的是"写 app 的 agent"还是"app 里的 agent"，这 5 条都成立。它们是整份文档的脊柱。

| # | 原则 | 含义 | 反面教材 |
|---|---|---|---|
| ① | **确定性外壳 + 非确定性内核** | 把不确定性收敛在尽可能小的核里（模型调用本身），核外的一切都做成确定的、可重放的。 | 让模型直接拼 SQL 并直连数据库执行 |
| ② | **契约 / 边界先行** | 先定输入输出契约、权限边界、"完成"的定义，再让模型往里填。契约是写给程序看的，不是写给人看的。 | 先让模型自由发挥，再回头补规则 |
| ③ | **模型提议，程序裁决** | 模型只负责"提议"（生成候选 SQL、候选答案、候选修改）；是否采纳由确定性代码裁决。 | 模型说能执行就执行、说对就当对 |
| ④ | **fail-closed（不确定退到安全侧）** | 任何歧义、解析失败、权限存疑，默认退到最安全的结果（拒绝/只读/降级为对话），而不是"赌一把"。 | 解析失败时猜一个默认值继续跑 |
| ⑤ | **可验证的反馈闭环** | 每一步都要有机器能判分的反馈：CI、单测、evals、线上观测。没有闭环的"改进"是错觉。 | 靠人肉手测判断"好像好了" |

一句话串起来：**①划出确定/不确定的边界，②③④规定边界上的行为，⑤保证边界本身在持续收紧。**

---

## 2. 方向一：用 AI 做 app（agentic coding）

核心命题：**怎么让一个会写代码、但会犯错的 agent，既跑得快又不跑偏。**

### 2.1 spec-first，而且要 scrutinize（审问）

- 先写 spec（PRD / tech spec / 验收标准），再让 agent 动手。spec 是 agent 的"意图来源"，没有它 agent 只能从聊天记录里猜。
- 但 spec-first 不等于"spec 写完就盲信"。**要 scrutinize**：自己读、让独立 reviewer 读、对着边界用例反问。agent 生成的 spec 往往看着合理、细看有洞。
- spec 与 code 要**锁步**：code 改了语义，spec 同步更新；spec 改了，code 跟上。两者漂移就是下一个 bug 的温床。

### 2.2 小步 PR 化，保护主干，关键检查设为必需

- 把大任务切成**小到可单独 review、可单独回滚**的 PR slice，而不是一个巨型 commit。
- **保护主干**：直接 push 到主分支的能力收掉，改动一律走 PR。
- **关键检查设为 required**：typecheck / lint / format / 单测 / smoke 设成合并的硬门槛——不是"建议跑"，是"不过就合不了"。

### 2.3 CI 即契约：把"对错"变成地面真值（ground truth）

> typecheck、lint、format、单元测试、smoke test —— 这一套就是 agent 的"地面真值"。

- agent 之所以**敢快**，正是因为有这套自动化的"对错判定器"在兜底：写错了 CI 立刻红，不用等人发现。
- 反过来，**CI 覆盖不到的地方，agent 就不该自信**。所以要持续把"靠人眼才能发现的错"往 CI 里搬。
- 这就是那句总结的前半句：**把「对错」机器化，agent 才敢快。**

### 2.4 诊断先于修复（埋点 → 观测 → 假设 → 验证 → 才动手）

这是整份文档里最反直觉、也最值钱的一条工程纪律。**agent（和人）最容易犯的错是看到症状就开始改。** 正确顺序是：先让系统可观测，再定位，最后才动手。

> **真实案例（evidata 的一个 bug 的逐层下钻）：**
>
> 1. **表面症状**：AI 返回的内容像是被 **截断（truncation）** 了。
> 2. 加日志观测后发现：不是截断，是模型吐出了 **非法 JSON**。
> 3. 再往下看原始响应：非法 JSON 的根因是模型一次发了 **多个 tool_call**，把结构撑坏了。
> 4. 继续追到传输层：多 tool_call 触发了上游 **HTTP 400**。
>
> 每一层都不是"猜"出来的，是**靠观测一层层定位**出来的。如果在第 1 层就动手"修截断"（比如调大 max_tokens），只会浪费时间且掩盖真因。

纪律固化为流程：**埋点 → 观测 → 形成假设 → 验证假设 → 才允许动手改。**

### 2.5 用 issue 驱动 backlog，按优先级从 board 拉活

- 不靠聊天记录记"接下来做什么"，而是把工作沉淀成 issue / board 条目，带优先级。
- agent 的工作方式变成：**从 board 按优先级拉一个 ready 的活 → 做 → 验证 → 关闭 → 再拉下一个。** 这让上下文持久化、可恢复、可被多方协作。

### 2.6 独立 review（subagent / Codex / 人）

- 写代码的 agent **不是**自己代码的合格 reviewer（它有自己的盲点和过度自信）。
- 引入**独立**的 review 视角：另一个 subagent、另一个工具（如 Codex）、或人。独立性是关键——同一个上下文里"自己审自己"抓不到真问题。
- 实践证明：**独立 review 确实抓到了自审漏掉的真 bug。**

### 2.7 线上真验，不止单测

- 单测过 ≠ 功能对。要在尽量接近真实的环境里跑端到端（smoke、真实数据库、真实模型 provider）。
- 模型相关的功能尤其如此：单测里 mock 掉模型，恰恰把最不确定的部分屏蔽了。

### 2.8 把「协作方式」本身也写进 repo —— 而不是靠 agent 的记忆

> **同一招再升一层：别让"我们怎么协作"只活在模型/会话的记忆里（会丢、不可移植、换工具就没）；把它放进 repo —— 可读、可共享、可强制、换 session 也跑不掉。**

**踩到的真实问题（evidata，2026-06-19）**：一整套高效协作约定——任务记成带优先级的 issue、PR 读 Codex 的 P0/P1/P2、里程碑才打断人决策、不推 main/rebase-only、spec 与代码同步、UI 改动真浏览器验证——**当时全都只存在 AI 助手的私有 memory 里**（`~/.claude/...`）。后果：**机器本地、工具私有、不进 git、不共享、memory 一重置就全没**。换 session / 换机器 / 来个新贡献者（或换个 AI 工具），这套工作方式**毫无保障**——这正是"靠记忆"的脆弱。

**做法（PR #107）**：搬进 repo，并尽量升级成"机制"而非"散文"：

| 层级 | 落地物 | 说明 |
|---|---|---|
| **散文（prose）** | `AGENTS.md` 的 "Collaboration & workflow" 段 | 跨工具的单一事实源（Codex 这类直接读 `AGENTS.md`，它正成为跨 agent 的事实标准） |
| **工具桥接** | `CLAUDE.md` 用 `@AGENTS.md` 导入 | Claude Code 默认读 `CLAUDE.md`；一行 import 让它**从 repo 加载约定**，不依赖私有 memory |
| **机制（enforcement）** | PR 模板 + issue 表单（优先级必填）+ 必需检查 + 分支保护 | 能机器强制的，就别只靠"读到/记住" |

**关键原则（harness engineering 的递归应用）**：
> **enforcement > prose。** 约定越是被"机制"承载（CI 必需检查、protect-main、label、模板），就越不依赖任何人/任何 agent 去"记得"。散文（AGENTS.md）是兜底，留给不好机器化的部分（决策门、验证纪律）。

**一个边界判断**：要分清**项目约定**（共享 → 进 repo）与**个人偏好**（私有 → 留 memory）。"对话用中文""学习笔记沉淀到 study 仓库"是维护者个人喜好，**不进共享 repo**；"rebase-only""读 Codex 评论"是任何贡献者都该守的项目约定，**必须进 repo**。

**为什么这条也算 harness**：整篇讲的是"别让模型自己搞定一切，把它放进一个跑不出去的盒子"。**这一条只是把同一招用在了'与 agent 协作的方式'本身上**——不让协作规则活在易失记忆里，而是固化进版本化、可强制、跨工具的 repo。可以叫 **meta-harness：连"怎么用 harness"都要被 harness 住。**

### 2.9 一句话总结

> **把「对错」机器化，agent 才敢快；把「意图」文档化，agent 才不跑偏；把「协作方式」repo 化，换谁来都不走样。**

CI/测试 = 机器化的"对错"（让它快）；spec/issue/decision-log = 文档化的"意图"（让它不偏）；AGENTS.md/CLAUDE.md/模板 = repo 化的"协作方式"（让它可移植、不丢）。三者缺一不可。

---

## 3. 方向二：把 AI agent 装进 app（product harness）

核心命题：**怎么让一个产品功能里的 agent，面对真实用户、真实数据、不可靠的模型，依然给出可信、安全、可恢复的结果。**

> **一句话定调：这一层的硬核功夫，几乎全在"对抗模型的不可靠"上。** 真正难的不是调通一次 happy path，而是处理它会以多少种方式坏掉。

### 3.1 分层 harness（从输入到横切）

| 层 | 职责 | evidata 里的具体做法 |
|---|---|---|
| **输入 / 意图层** | 在模型介入前先做确定性判断，分流意图 | IME 守卫（避免中文输入法组字途中误触发）；问候/闲聊的启发式短路；把意图**工具化**为离散动作：`reply`（对话）vs `draft_sql`（生成查询） |
| **决策循环** | 控制模型的多步行为、保证收敛 | tool-use loop；**force-finalize**（到点强制收口、产出最终答案）；**step budget**（步数预算，防止无限绕圈） |
| **工具与权限边界** | 划定模型"能做什么"的物理边界 | SQL 安全门**只读**（拦写操作）；**fail-closed**（拿不准就拒）；**唯一 DB 路径**（所有数据库访问只走这一条受控通道，杜绝绕行） |
| **执行 + 证据** | 真正执行，并留下可审计的证据 | 结果**脱敏**后再回传；**每条查询都记录（G4）**——可审计、可复盘、可计费 |
| **修复 + 校验** | 模型输出坏了怎么救 | **JSON 修复**（容忍常见畸形）；**契约校验**（结构不符直接判废）；**re-prompt-once**（带着错误信息让模型重来一次，且只一次） |
| **横切** | 贯穿所有层的工程韧性 | 端到端**取消**（AbortSignal 一路传穿，用户中断即真停）；**瞬时错误重试**；**token 计量**（成本与配额可观测） |

### 3.2 输出契约：把"可信"变成可机器校验的东西

这是方向二最精华的一条。模型会"自信地胡说"，所以**不能让模型自己声称"我说的是对的"**，要用契约逼它拿出依据：

> **G3 守则：模型若要对数据下断言（"退款总额是 X"、"最大的客户是 Y"），必须有据——即必须真的执行过查询、拿到过结果。**
>
> - **有据** → 允许作为 **Answer**（对数据的正式回答，可呈现给用户当结论）。
> - **无据** → 只能降级为 **Message**（普通对话内容，不得伪装成数据结论）。

这就是 **Answer vs Message 的二分**：用一个确定性的契约，把"基于真实数据的结论"和"模型的自由发挥"在类型层面分开。用户看到的"答案"永远有查询证据撑着；模型想空口断言，最多只能落到"对话"里。

> 这正是原则 ③（模型提议、程序裁决）+ ④（fail-closed）在产品层的落地：模型提议一个断言，程序检查它有没有证据，没有就降级——而不是默认相信。

### 3.3 对抗不可靠的四件套

1. **输出契约**：如上，结构化、可校验、分级（Answer/Message、G3/G4）。
2. **防御式解析**：永远假设模型输出是畸形的——JSON 可能多逗号、缺括号、夹散文、一次多 tool_call。先修复、再校验、校验不过就判废，绝不"将就着用"。
3. **确定性兜底**：模型彻底失败时，要有不依赖模型的退路（固定话术、降级为只读对话、明确报错），而不是把异常透传给用户或卡死。
4. **韧性重试**：区分**瞬时错误**（网络抖动、429、上游 5xx → 退避重试）和**确定性错误**（契约不符 → re-prompt-once，不无脑重试）。重试要有上限，配合 step budget 和取消。

---

## 4. 对标范式：Claude / OpenAI 这类团队怎么想

把上面两个方向的零散实践，对到业界 harness engineering / evals 驱动的范式上。

### ① evals 驱动开发（最该提前做的一件事）

- **先建评测集**：把"什么算对"沉淀成一组可重放的样例（输入 + 期望/判分标准）。
- **自动判分**：能用断言判分就断言；判不了的（答案质量、是否答非所问）用 **LLM-as-judge**。
- **每次改动都跑回归**：改 prompt、改工具、换模型——都对评测集跑一遍，看指标涨跌。
- **盯的核心指标**：成功率、平均步数、token 消耗、拒答率（以及延迟、成本）。
- **为什么最该提前**：evals 是方向二的"CI"。没有它，你就是在用手测当回归测试——慢、不全、抓不到回归。这是本次实践最大的"如果重来会更早做"的事（见第 5 节）。

### ② agent loop 的克制：能确定性编排，就别上 autonomous agent

- 每加一步"让模型决定"，就多一份不确定性和成本。**先问一句：这一步，真的必须由模型决定吗？**
- 能用确定性 **workflow**（固定的步骤编排）解决的，就别上**自主 agent**（模型自己决定下一步）。
- 典型分层：意图分类、参数校验、安全检查 → 确定性代码；只有"需要语言理解/生成"的那一小步 → 模型。这正好回扣原则①（把不确定性收敛到最小的核）。

### ③ 工具设计 = API 设计

- 给模型的 tool，本质是一套 API：命名、参数、描述、错误信息都要像设计公共 API 一样讲究。
- 工具要**窄而清晰**（一个工具干一件事）、**难以误用**（约束放在 schema 里）、**返回结构化**（让程序能裁决，而不是让模型再去解析自然语言）。evidata 把意图做成 `reply` / `draft_sql` 两个离散工具，就是这个思路。

### ④ context engineering

- 模型只看得到你喂进上下文的东西。**喂什么、怎么排、放多少、何时裁剪**，直接决定输出质量。
- 实践要点：相关的放进来、无关的拦在外、长对话要压缩/摘要、把"地面真值"（schema、约束、证据）显式放进上下文而不是指望模型记得。

### ⑤ guardrails / fail-closed / human-in-the-loop

- **guardrails**：输入侧（拦注入、拦越权意图）+ 输出侧（拦敏感数据、拦无据断言）双向护栏。
- **fail-closed**：护栏拿不准时默认拒绝/降级（原则④）。
- **human-in-the-loop**：高风险动作（写数据库、花钱、改 scope）必须人确认，agent 不自作主张。

### ⑥ 可观测 + 红队

- **可观测**：每一步的输入输出、token、耗时、决策路径都要可追溯（呼应 G4、诊断先于修复）。
- **红队**：主动构造对抗输入（注入、诱导越权、诱导无据断言），在上线前找漏洞，而不是等用户找到。

### 推荐读物（已核对｜截至 2026-06-18）

> 本节链接与要点经一次 deep-research 核对（fan-out 搜索 → 抓取官方源 → 对每条断言做 3 票对抗式核验；共 84 条断言，**0 条被推翻**）。优先选官方源。下面先列几个"会让你找错文档/用错术语"的最新变化。

#### ⚠️ 几个已核对的"currency flags"（容易踩的坑）

- **Anthropic 文档站已迁到 `platform.claude.com`** —— 旧的 `docs.anthropic.com` 链接多会跳转或失效，收藏请用新域名。
- **OpenAI 开发者文档在 `developers.openai.com`** —— 不再是 `platform.openai.com/docs` 的老路径。
- **OpenAI 主推 Responses API（agentic by default）** —— 它是 Chat Completions 的演进，单次请求内就能跑 agent loop、调多个内置工具（web/file search、computer use、code interpreter、远程 MCP）；Chat Completions 仍支持、未废弃，但新项目建议用 Responses。迁移指南：<https://developers.openai.com/api/docs/guides/migrate-to-responses>
- **"function" 现在是 "tool" 的子类型** —— OpenAI 把旧的 "function calling" 收进更广的 "tools" 框架（未显式废弃，但术语已更新）。
- **OpenAI 旧 Evals 平台将于 2026-11-30 关停**，新用户被引导到 Datasets —— 但"怎么做 evals"的方法论仍然有效。

#### Anthropic

**《Building Effective Agents》（构建高效 agent）** —— <https://www.anthropic.com/research/building-effective-agents>（2024-12-19）
- **workflow**（用预定义代码路径编排 LLM+工具）vs **agent**（LLM 动态自主决定流程与工具）的权威区分——本篇范式②的来源。
- 核心启发式：**从简单开始，只在简单方案不够时才加 agentic 复杂度**；workflow 适合要可预测的明确任务，agent 适合需要灵活与模型自主决策的规模化任务。
- 五种可组合 workflow 模式：prompt chaining、routing、parallelization、orchestrator-workers、evaluator-optimizer。
- **ACI（agent-computer interface）要像 HCI 一样下功夫**：工具描述当 docstring 写、跨大量样例测模型怎么用工具、用 poka-yoke（防呆）设计降低误用。

**《Effective context engineering for AI agents》（上下文工程）** —— <https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents>（2025-09-29）
- 区分 context engineering（curate/maintain 推理时的最优 token 集）与 prompt engineering（写指令）；前者是面向多轮、长任务 agent 的更大学科。
- **"context rot"**：token 越多，模型从上下文准确召回的能力越差——上下文是有限资源。
- 长任务上下文管理四件套：just-in-time 检索（用轻量标识符 / glob、grep、read_file，而非预算 embeddings）、compaction（近上限时摘要历史）、结构化记笔记（窗口外持久记忆）、子 agent（返回 1k–2k token 浓缩摘要）。
- 总原则：找"最小的高信号 token 集"；系统提示要在"硬编码逻辑"与"模糊指导"之间找对 altitude。

**《How we built our multi-agent research system》（多 agent 系统复盘）** —— <https://www.anthropic.com/engineering/multi-agent-research-system>（2025-06-13）
- orchestrator-worker 架构：lead agent 定策略、把不同子问题并行委派给 subagent，再综合。
- 效果：Opus 4 当 lead + Sonnet 4 当 subagent，比单 Opus 4 在内部研究评测上高 **90.2%**。
- **代价警告**：token 用量解释了约 **80%** 的性能方差；agent 比 chat 多约 4×、多 agent 约 15× token——**只对高价值任务才值得**。
- 委派要给 subagent 明确的目标 / 输出格式 / 工具与来源指引 / 任务边界，并按复杂度配资源（简单事实查找 = 1 agent + 3–10 次工具调用）；评测可从约 **20 条代表性 query + 单次 LLM-as-judge（0–1 分 + pass/fail）** 起步，但人测仍不可少。

**《Writing effective tools for AI agents》（给 agent 写工具）** —— <https://www.anthropic.com/engineering/writing-tools-for-agents>（2025-09-11）
- **工具不是越多越好**；常见错误是把每个 API endpoint 包成一个工具。应合并成高价值"工作流工具"（一个 `schedule_event` 取代 `list_users`/`list_events`/`create_event`）。
- 迭代式建工具：本地原型（包成本地 MCP server / 桌面扩展，在 Claude Code/Desktop 里测）→ 基于真实用法生成大量评测任务 → 用 Claude 分析 transcript 再重构。
- 工具只返回高信号、相关、token 高效的信息（分页 / 过滤 / 截断 / `response_format` 的 concise vs detailed）；按 `service_resource` 前缀给工具命名空间（`asana_search`…）。
- 微调工具描述 + 无歧义参数命名带来大幅提升（精确描述助 Sonnet 3.5 拿下 SWE-bench Verified SOTA；参数用 `user_id` 而非 `user`）。

**Claude Docs · Tool use（工具使用）** —— <https://platform.claude.com/docs/en/docs/agents-and-tools/tool-use/overview>
- 权威术语是 **"tool use"**：Claude 返回结构化调用，你的 app 执行（client tools）或 Anthropic 执行（server tools）。
- tool-use 循环：Claude 以 `stop_reason=tool_use` + tool_use block 回应 → 你执行 → 回传 `tool_result`；server tool 在 Anthropic 侧跑、直接拿结果。
- 调用与否由工具描述决定，可用系统提示 steer（"Use the tools to investigate before responding." 实测能提高工具使用率）；`strict:true` 保证调用严格符合 schema。

**Claude Docs · 评测工具 / test & evaluate** —— <https://platform.claude.com/docs/en/docs/test-and-evaluate/eval-tool>
- Console 内置 Evaluate 工具，跨多场景测 prompt；prompt 用 `{{变量}}` 双花括号（至少 1–2 个）。
- 测试用例三种来源：手动加行、Claude 自动生成、CSV 导入；支持多 prompt 并排对比、5 分制质量打分、prompt 版本化重跑。
- 更新动向：Anthropic 还有更新的 evals 文章——《Demystifying evals for AI agents》（2026-01-09）、《Designing AI-resistant technical evaluations》（2026-01-21），在 Engineering 博客。

#### OpenAI

**《A Practical Guide to Building Agents》（PDF）** —— <https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf>（2025-04-07；落地页 <https://openai.com/business/guides-and-resources/a-practical-guide-to-building-ai-agents/>）
- agent 定义：**用 LLM 控制 workflow 执行**的系统；明确排除 chatbot / 单轮 LLM / 情感分类这类"接了 LLM 但不由它控流程"的应用。
- 三要素框架：**Model + Tools + Instructions**；先用最强模型建 baseline，再在 evals 仍通过的地方换小模型省成本/延迟。
- 工具分三类：**Data、Action、Orchestration（agents-as-tools）**；工具定义要标准化、可复用，支持多对多。
- **先把单 agent loop 做到极致再上多 agent**；两种编排：Manager（中心 agent 用工具调度专家）与 Decentralized（对等 agent 单向 handoff）。工具过载源于相似/重叠而非数量（有团队 15+ 工具没事，有的不到 10 个重叠工具就乱）。
- guardrails 是**分层防御**（LLM 判别 + 规则/正则 + Moderation API），对超过失败阈值或高风险/不可逆动作（大额退款、付款）触发人工介入。

**OpenAI Evals 指南（"Working with evals"）** —— <https://developers.openai.com/api/docs/guides/evals>
- ⚠️ **正在被取代**：旧 Evals 平台关停日 2026-11-30，新用户被导向 Datasets；但下面的方法论仍有效。
- 三步法：把任务描述成 eval（`data_source_config` + `testing_criteria`）→ 对测试输入跑 → 看结果并迭代 prompt。
- 用 JSONL 测试数据建评测集，每条符合 JSON schema、含人工标注的 **ground truth** 作正确性基准；升级/换模型时 evals 尤为必要。

**OpenAI Graders（判分器）** —— <https://developers.openai.com/api/docs/guides/graders>
- 五种 grader：String Check、Text Similarity、**Score Model（LLM-as-judge）**、Python、Multigraders。
- 建议 grader 产出**连续/平滑分**而非二元 pass/fail，好让优化过程看出哪些改动有效。
- 当代码判分不足以评价开放式回答时才用 LLM-as-judge，并用多个候选回答验证 grader 的稳定性。

**OpenAI Function calling（≈ tool calling）** —— <https://developers.openai.com/api/docs/guides/function-calling>
- 术语更新：现在把 "function" 当作 "tool" 的子类型（由 JSON schema 定义的一种工具）。
- 五步循环：带 tools 发请求 → 收到 tool call → 执行你的应用代码 → 回传 tool 输出 → 收最终回复或继续调用。
- 最佳实践：清晰的函数名/参数描述/指令，用 enum + 对象结构让非法状态无法表达；软上限"一轮开始时 < 20 个函数"，别让模型填你已知的参数、把总连用的函数合并。

**OpenAI Agents SDK** —— <https://developers.openai.com/api/docs/guides/agents> · Python 参考 <https://openai.github.io/openai-agents-python/>
- 核心原语：**Agents、Handoffs、Guardrails** + Sessions、Tools、Tracing；内置 agent loop 自动处理工具调用并迭代到完成。
- 工具 = 把 Python 函数用 `@function_tool` 装饰，自动按签名 + docstring 生成 JSON schema（inspect + Pydantic 校验）；MCP server 工具同理。
- 两种多 agent 编排：handoffs（对等委派）与 manager 式（agents-as-tools，用 `.as_tool()` 暴露）；区分 hosted 工具（WebSearch/FileSearch/CodeInterpreter，在 OpenAI 侧跑）与本地函数工具。
- 默认对 OpenAI 模型用 **Responses API** 而非 Chat Completions，体现 "tools" 框架取代旧 function-calling。

---

## 5. 诚实反思：哪里做得好，哪里能更好

### ✅ 做得好

- **spec-first + scrutinize**：先定意图再动手，且不盲信生成的 spec——靠审问挡掉了一批方向性错误。
- **诊断先于修复**：truncation → 非法 JSON → 多 tool_call → HTTP 400 那条下钻链，靠观测逐层定位，没有在症状层瞎改。
- **fail-closed 一以贯之**：从 SQL 只读门到契约校验到意图分流，"拿不准就退到安全侧"是稳定的默认行为，不是个别地方的临时处理。
- **契约把"可信"变成可机器校验**：G3（有据才算 Answer）/ G4（每查询留证）/ Answer vs Message，把"这答案可不可信"从主观判断变成了类型层面的硬约束。
- **独立 review 抓到真问题**：引入独立视角（subagent / Codex / 人）确实发现了自审漏掉的 bug，验证了"不能自己审自己"。

### ⚠️ 能更好

- **evals 应该更早**：太多 bug 是靠人肉手测发现的。如果一开始就建评测集 + 自动判分，这些回归本该被机器在更早、更便宜的时点抓到。这是最大的改进点。
- **个别 PR 偏大**：有些改动一口气塞太多，违背了"小到可单独 review/回滚"的原则，增加了 review 难度和回滚成本。
- **有些设计决策实现到中途才定**：部分边界（契约形态、降级策略）是在写的过程中才拍板的，更理想的是契约/边界先行（原则②），实现阶段只填不议。

---

## 6. 一页 checklist + 学习路径

### ✅ 上手前过一遍的 7 条 checklist

1. **先定契约 / 边界** —— 输入输出契约、权限边界、"完成"的定义，写在动手之前。
2. **把"对错"机器化** —— typecheck / lint / 单测 / smoke / evals，设为必需，当地面真值。
3. **模型只提议，程序裁决** —— 模型生成候选，确定性代码决定是否采纳。
4. **fail-closed** —— 歧义、解析失败、权限存疑，一律退到最安全的结果。
5. **防御式处理模型输出** —— 假设输出畸形：先修复、再校验、校验不过判废、必要时 re-prompt-once。
6. **预算 + 可取消 + 可观测** —— step budget、AbortSignal 端到端、token 计量、每步可追溯。
7. **evals 当回归，越早越好** —— 评测集 + 自动判分（含 LLM-as-judge），每次改 prompt/工具/模型都跑。

### 📚 学习路径（由浅入深、边学边做）

1. Anthropic《Building Effective Agents》—— 先建立"workflow vs agent、何时该上 agent"的判断框架。
2. OpenAI《A Practical Guide to Building Agents》—— 补全 loop / guardrails / 工具设计的工程实操。
3. **两家的 Evals 文档，并动手建一个自己的评测集** —— 这一步最关键，别只读不做。
4. Tool use / function calling —— 把"工具即 API"落到代码。
5. Context engineering + multi-agent 复盘 —— 学会喂对上下文、理解多 agent 的坑。
6. Red-teaming —— 学会主动攻击自己的 agent，在上线前找洞。

---

> **最佳内化方式：把上面这份 checklist 落成你自己项目里的一个 eval harness——边学边在自己的项目上跑通一遍。读懂不算会，跑通才算。**
