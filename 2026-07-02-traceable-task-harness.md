# Traceable Task Harness:从 PRD 到 agent 可执行任务

> 副标题:从 0 用 Codex / Claude Code / 其他 LLM agent 做 app 时,真正缺的往往不是更长的 PRD,而是把意图拆成**可追溯、可执行、可验证**的小任务的外壳。PRD 负责说"为什么和要什么",Traceable Task Harness 负责让 agent 每次只拿到"此刻该做什么、依据哪里、怎样算完成"。
>
> 日期:2026-07-02 ｜ 来源:一次关于"从 0 vibe coding 产品容易中途失控"的讨论,结合 OpenAI Codex、OpenAI Agents、Anthropic Claude Code、Anthropic harness / context engineering 相关官方材料。
>
> 配套阅读:[Harness Engineering](2026-06-18-harness-engineering.md)(总论)、[从意图到产品](2026-06-20-product-judgment.md)(产品层判断)、[交付外壳](2026-06-20-delivery-harness.md)(CI / review / 发布门控)、[多 agent 工作流](2026-06-20-agentic-multi-agent-workflows.md)(并行协作)。
>
> 适用对象:用 coding agent 从 0 做产品的人。尤其适合这种状态:PRD / tech spec / data model / UI 设计都写了,但做到中间发现产品行为、交互和代码产出开始偏离想象。

---

## 摘要

本文不是严格意义上的学术论文,而是一篇带论文式骨架的工程学习笔记。它讨论一个实践问题:为什么 coding agent 在成熟 codebase 上修 bug、加功能、做 investigation 往往效果很好,但从 0 构建产品时容易中途失控。核心结论是:成熟 codebase 自带架构、测试、命名、CI、历史代码等隐形 harness;从 0 项目缺少这层外壳,所以 PRD、tech spec、data model、UI design 即使写得很详细,也不能自动变成 agent 可执行、可验证的小任务。

本文提出的实践框架是 **Traceable Task Harness**:用稳定 Source ID 把 PRD / flow / data model / UI contract 中的关键事实索引起来,再把 milestone 编译成带 Context Pack、Acceptance Criteria、Verification Command、Progress State 的小任务。这样 agent 每次工作时只读取当前任务所需的高信号上下文,并通过明确验收方式判断是否完成。

**研究问题**:如何把从 0 产品开发中的大块产品文档,转化为 coding agent 能稳定执行的小任务,同时避免上下文污染、遗漏关键约束和"看起来完成"的幻觉?

**核心主张**:从 0 vibe coding 的关键不是写更长的 PRD,而是建立一层把产品意图翻译成可追溯任务和可验证反馈的 harness。

**主要贡献**:

1. 区分成熟 codebase 的隐形 harness 与从 0 项目的 harness 真空。
2. 将"关键词关联上下文"升级为稳定 Source ID + Context Pack。
3. 给出从 product brief、requirements、flows、data model、UI contracts 到 feature list / issue 的最小文件结构。
4. 给出 issue 模板、prompt 模板和对抗性检查,让这套方法能直接落地。

**适用边界**:这不是大型组织的完整需求管理方法,也不是证明某种 agent 开发范式优于另一种的实证研究。它适用于单人或小团队用 Codex / Claude Code 等 coding agent 从 0 做 MVP,目标是用最小流程约束换取更稳定的上下文、验收和产品方向。

---

## 0. 一句话定位

> **成熟 codebase 自带隐形 harness,从 0 项目没有。所以从 0 vibe coding 的关键步骤,是把大文档编译成一组带 source id、context pack、验收标准和验证命令的任务。** 这不是项目管理仪式,而是给 agent 的上下文加载器和完成判定器。

一句更短的版本:

> **人负责意图和取舍,agent 负责执行,harness 负责记忆、边界和验收。**

---

## 1. 为什么成熟项目好用,从 0 项目容易失控

成熟项目里,agent 工作得好,不是因为它突然更聪明,而是因为项目已经给它提供了大量"地面真值":

| 成熟项目里的隐形外壳 | 它给 agent 的信号 |
|---|---|
| 目录结构 / 模块边界 | 代码应该放哪里,不该跨哪条边界 |
| 既有类型 / schema / helper | 什么抽象已经存在,什么不要重造 |
| 测试 / CI / lint | 怎样算没破坏系统 |
| 历史代码风格 | 同类问题以前怎么写 |
| issue / PR 历史 | 产品行为背后的取舍 |
| 用户已跑过的真实路径 | 哪些功能真的重要 |

从 0 项目里,这些都没有。于是 PRD 再长,agent 仍然会遇到三个真空:

1. **上下文真空**:当前任务到底应该读 PRD 的哪几段、tech spec 的哪几段、UI 设计的哪几个状态。
2. **完成真空**:写完代码之后,到底用什么自动或半自动方式证明它符合产品意图。
3. **取舍真空**:当 PRD 内部有模糊处或冲突时,哪些决定必须回到人,哪些可以让 agent 自行补全。

所以"从 0 做产品"的失败,常常不是 coding failure,而是 **harness failure**:agent 没有稳定方式把大文档加载成此刻的小上下文,也没有稳定方式判断自己真的做对了。

---

## 2. 对原直觉的评审:方向正确,但关键词不够硬

原直觉:

> 把 milestone 拆成细粒度 Jira ticket / GitHub issue,每个任务关联 PRD、tech spec、data model、user story 中对应部分,让 agent 既不漏信息,也不把无关内容塞进 context。

这个判断是对的,而且非常接近 Anthropic 对 long-running coding agents 的 harness 思路:先初始化环境、拆 feature list,后续每个 agent session 做增量进展,读 progress,结束时留下结构化更新。Anthropic 明确指出,高层 prompt 直接让 agent 长时间造 app,容易一口气做太多、上下文耗尽、半成品无人解释,或者后续 session 看到有进展就过早宣布完成。

但这套想法要升级三点:

| 原方案 | 问题 | 更硬的版本 |
|---|---|---|
| 用"关键词"关联上下文 | 关键词会漂移、同义词会漏召回、相似词会误召回 | 用稳定 `Source ID`,关键词只做辅助搜索 |
| issue 写"做什么" | agent 容易实现一个看似相关的功能 | issue 必须写 user-visible behavior 和 acceptance criteria |
| 让 agent 自己判断完成 | 生成者自评偏乐观,UI/交互尤其明显 | 每个任务至少有一个 verification command 或 evaluator |

所以最终形态不是"更细的 Jira",而是:

> **Traceable Task Harness = Source IDs + Context Pack + Acceptance Criteria + Verification Command + Progress State。**

---

## 3. 核心心智模型:四次翻译

从产品直觉到可交付 app,中间要翻译四次:

| 阶段 | 产物 | 失败时的症状 |
|---|---|---|
| 意图 → 契约 | PRD / non-goals / flows / data model | agent 用默认产品想象填空 |
| 契约 → Source IDs | `REQ-*` / `FLOW-*` / `DATA-*` / `UI-*` / `NFR-*` | 文档很多,但无法稳定引用 |
| Source IDs → Task | issue / ticket / feature list | 任务看似清楚,实际漏掉关键约束 |
| Task → Verification | test / smoke / browser check / human review | "完成"只是 agent 的主观声明 |

这四次翻译缺任何一层,都会回到"做到中间才发现不对"。

最重要的一点:**ticket 不是把 PRD 切碎,而是把 PRD 的相关切片编译成一个可执行合约。**

---

## 4. 最小文件集:别先搭平台,先搭这几个文件

一个从 0 app 的最小 harness,够用就这些:

```text
docs/
  product-brief.md
  requirements.md
  flows.md
  data-model.md
  ui-contracts.md
  decisions/
    ADR-0001-*.md
feature_list.json
progress.md
AGENTS.md
CLAUDE.md
scripts/
  check
  smoke
```

### 4.1 `product-brief.md`

一页就够。超过一页时,通常是把还没拍板的问题伪装成了结论。

```md
# Product Brief

## 用户是谁

## 核心 job-to-be-done

## MVP 成功标准

## 明确不做

## 最大风险

## 需要人类拍板的问题
```

### 4.2 `requirements.md`

每条需求给稳定 ID。ID 是给 agent、issue、测试、commit、PR 串起来用的。

```md
## REQ-AUTH-001:用户可以用邮箱登录

用户输入邮箱和验证码后进入工作区。

Acceptance:
- 验证码错误时停留在登录页并显示错误。
- 登录成功后创建 session。
- 刷新页面后仍保持登录。

Related:
- FLOW-AUTH-001
- DATA-USER-001
- UI-AUTH-001
```

### 4.3 `flows.md`

flow 写用户路径,不是组件结构。

```md
## FLOW-AUTH-001:邮箱验证码登录

1. 用户打开登录页。
2. 输入邮箱。
3. 收到验证码。
4. 输入验证码。
5. 成功进入 workspace。

Edge states:
- 邮箱格式错误。
- 验证码错误。
- 验证码过期。
- 网络失败。
```

### 4.4 `data-model.md`

data model 要把不变量写清楚。agent 很容易只建字段,不建约束。

```md
## DATA-USER-001:User

Fields:
- id
- email
- created_at

Invariants:
- email globally unique.
- email is normalized before persistence.

Related:
- REQ-AUTH-001
```

### 4.5 `ui-contracts.md`

UI design 不只放截图,要写状态。状态比截图更能约束 agent。

```md
## UI-AUTH-001:登录页

States:
- empty
- email_invalid
- code_sent
- code_invalid
- loading
- success_redirecting

Non-goals:
- MVP 不支持第三方 OAuth。
```

### 4.6 `feature_list.json`

Anthropic long-running harness 里的关键思路是让 agent 面向 feature list 增量推进,不要靠聊天记录记进度。可以用 JSON,也可以用 Markdown 表格。JSON 的好处是后续能被脚本检查。

```json
[
  {
    "id": "FEAT-AUTH-001",
    "source_ids": ["REQ-AUTH-001", "FLOW-AUTH-001", "DATA-USER-001", "UI-AUTH-001"],
    "description": "User can sign in with email verification code",
    "priority": "P0",
    "status": "ready",
    "verification": "scripts/smoke auth-email-login",
    "passes": false
  }
]
```

### 4.7 `AGENTS.md` / `CLAUDE.md`

`AGENTS.md` 是 Codex 的 repo guidance 入口,适合放项目结构、测试命令、工程约束、done definition。Claude Code 侧用 `CLAUDE.md` 存项目记忆和约定。为了跨工具,推荐:

```md
# CLAUDE.md

Read @AGENTS.md first.
```

这样共享约定只维护一份。工具专属细节再放各自文件。

---

## 5. Source ID 体系:关键词可以搜,不能当锚

推荐 ID 前缀:

| 前缀 | 含义 | 示例 |
|---|---|---|
| `REQ-*` | 产品需求 | `REQ-BILLING-001` |
| `FLOW-*` | 用户流程 | `FLOW-ONBOARD-002` |
| `DATA-*` | 数据模型 / 不变量 | `DATA-WORKSPACE-001` |
| `UI-*` | UI 状态 / 交互契约 | `UI-CHAT-003` |
| `NFR-*` | 非功能要求 | `NFR-PERF-001` |
| `ADR-*` | 架构 / 产品决策 | `ADR-0004-use-sqlite-first` |

规则:

1. **一个 ID 只表达一个稳定事实**。不要让 `REQ-AUTH-001` 同时覆盖登录、注册、邀请。
2. **issue 引用 ID,不是引用整份文档**。agent 根据 ID 读相关段落。
3. **ID 可以被废弃,不要复用**。需求改了就新建或标 superseded,避免历史 PR 语义漂移。
4. **每个 P0 feature 至少关联一个 REQ、一个 FLOW、一个 UI 或 DATA**。否则它很可能不是完整用户行为。

---

## 6. Issue 模板:一个任务就是一个可执行合约

````md
## Source IDs

- REQ-...
- FLOW-...
- DATA-...
- UI-...

## Goal

一句话说明这次要让用户能做什么。

## User-visible behavior

用户实际看到的行为,包括空状态、错误状态、成功状态。

## Context pack

只贴和本任务有关的文档摘录或链接:
- requirements.md#REQ-...
- flows.md#FLOW-...
- data-model.md#DATA-...
- ui-contracts.md#UI-...

## Acceptance criteria

- [ ] ...
- [ ] ...
- [ ] ...

## Verification

Run:
```bash
scripts/check
scripts/smoke ...
```

## Out of scope

明确不做什么,防止 agent 顺手加东西。

## Human decision required

列出必须由人拍板的问题。没有就写 None。
````

这里最关键的是 `Context pack` 和 `Verification`。没有这两个字段,issue 只是愿望清单。

---

## 7. 从 0 vibe coding 一个 app 的流程

### 7.1 第 0 阶段:先让 agent 采访你

不要一上来让 agent 写代码。先让它做产品采访:

```text
先不要写代码。请采访我,目标是产出 product-brief.md、requirements.md、flows.md、data-model.md、ui-contracts.md 的第一版。

要求:
- 主动指出含糊处。
- 把单向门决策列出来。
- 每条需求生成稳定 Source ID。
- 对每个 non-goal 给 2 个漂移信号。
```

这一阶段的输出不是"正确答案",而是让你的模糊直觉变成可审查材料。

### 7.2 第 1 阶段:walking skeleton

先做一条最薄的端到端路径:

- app 能启动。
- 有首页或主工作区。
- 有最小数据存取或 mock 数据。
- 有 `scripts/check`。
- 有一个 smoke test 或手动可跑的验证脚本。
- 有 `AGENTS.md` 写明 done definition。

别先铺满功能。没有 skeleton,后面每个 agent 都会自己发明启动方式、测试方式和目录结构。

### 7.3 第 2 阶段:把 PRD 编译成 feature list

让 agent 做这件事,但你要 review:

```text
基于 docs/ 下的文档,生成 feature_list.json。

要求:
- 每个 feature 是一个用户可见行为。
- 每个 feature 引用 source_ids。
- 每个 feature 有 verification。
- 标 P0/P1/P2。
- 标 blocked/ready。
- 如果某 feature 无法验证,把它拆小或标 human-review。
```

对抗性检查:

- 如果一个 feature 需要一次改 8 个模块,它太大。
- 如果一个 feature 只写"实现后端接口",它可能不是用户行为。
- 如果一个 feature 没有失败状态,它少了一半。
- 如果一个 feature 没有 out of scope,agent 会顺手扩展。

### 7.4 第 3 阶段:一次只做一个 ready feature

给 coding agent 的任务模板:

```text
Read AGENTS.md, feature_list.json, progress.md, and the source_ids for FEAT-...

Implement exactly FEAT-...

Constraints:
- Do not implement out-of-scope features.
- If requirements conflict, stop and report the conflict.
- Keep the smallest working diff.

Done when:
- Acceptance criteria pass.
- scripts/check passes.
- The listed verification passes.
- feature_list.json and progress.md are updated.
```

这和 OpenAI Codex 的 prompting 建议一致:prompt 里明确 Goal、Context、Constraints、Done when。Codex 官方也建议复杂任务先 plan、拆成更小步骤,并把稳定指导放进 `AGENTS.md`。

### 7.5 第 4 阶段:独立验证

写代码的 agent 不该自己当最终 reviewer。至少做一种独立验证:

- 另一个 agent 用 fresh context review diff。
- browser automation 跑用户路径。
- 人类试用最关键路径。
- 对开放式体验,让 reviewer 对照 rubric 打分。

Claude Code 官方 best practices 也建议用 subagent 研究或 review,因为它在独立 context 里工作,不会污染主会话。对于 UI/交互,独立 browser check 的价值更高,因为编译通过不等于体验正确。

### 7.6 第 5 阶段:milestone 不是合并点,是重新判断点

每个 milestone 结束时不要只问"做完了吗",要问:

- 这个产品现在的身份和 product brief 一致吗?
- 哪些 non-goals 出现了漂移信号?
- 哪些需求做到一半才发现其实定义错了?
- 哪些本该自动验证的东西还在靠人眼?
- feature_list 里是否有太多技术任务,缺少用户行为任务?

milestone 的作用是校正产品判断,不是攒一堆 PR。

---

## 8. Context 策略:给 agent 最小高信号上下文

Anthropic 的 context engineering 文章把问题说得很清楚:agent 的 context 是有限资源,工程问题不是"尽量塞更多",而是"在每次推理时放入最有用的一组 token"。从 0 app 的上下文策略可以这样落地:

| 场景 | 放进上下文 | 不放进上下文 |
|---|---|---|
| 做一个 issue | source_ids 对应段落、相关文件、验收标准、验证命令 | 整份 PRD、所有 UI 截图、无关历史讨论 |
| 做架构决策 | product brief、相关 ADR、关键 non-goals、约束 | 具体组件实现细节 |
| 做 UI 交互 | UI states、flow、截图或设计链接、可访问性要求 | 后端内部实现长文 |
| 做数据模型 | DATA 段落、不变量、迁移约束、调用方 | 视觉设计细节 |

三个实践原则:

1. **先索引,再加载**。Source ID 是索引,context pack 是加载结果。
2. **先摘要,再原文**。issue 里放短摘要,需要时再读原文段落。
3. **先当前任务,再全局背景**。全局背景放 `AGENTS.md`,任务细节放 issue。

---

## 9. 对抗性检查:这套方法会怎么坏

### 9.1 伪精细化

症状:issue 很多,但每个都写"实现 X 模块"。

修法:每个 P0/P1 issue 必须写 user-visible behavior。纯技术任务要挂到某个用户行为下面,或标明它解锁哪个 feature。

### 9.2 Source ID 通胀

症状:每句话一个 ID,agent 反而不知道重点。

修法:只有会被 issue、测试、PR 反复引用的稳定事实才配 ID。临时想法放 notes,不要进 ID 系统。

### 9.3 验收标准变散文

症状:acceptance 写"体验顺滑、逻辑正确"。

修法:改成可观察状态:

- 点击 X 后出现 Y。
- 输入非法 Z 后显示错误 A。
- 刷新后状态仍为 B。
- 运行 `scripts/smoke foo` 返回 0。

### 9.4 agent 被局部任务带偏

症状:每个 issue 都完成了,但整体产品不像原意。

修法:milestone 需要 product review,不是只看 issue close rate。每周跑一条真实端到端路径,对照 product brief 和 non-goals。

### 9.5 evaluator 太弱

症状:agent 总能让测试过,但人用起来不对。

修法:为 UI/体验加独立 browser check 和人类 rubric。代码测试负责"能不能",产品 review 负责"是不是"。

---

## 10. 一页落地清单

- [ ] 写一页 `docs/product-brief.md`,包含用户、JTBD、MVP 成功标准、non-goals、最大风险。
- [ ] 把 PRD 拆成带稳定 ID 的 `REQ-* / FLOW-* / DATA-* / UI-* / NFR-*`。
- [ ] 建 `feature_list.json`,每个 feature 引用 source_ids,有 priority/status/verification。
- [ ] 建 `AGENTS.md`,写 repo 结构、命令、约束、done definition。
- [ ] 建 `CLAUDE.md`,至少导入或指向 `AGENTS.md`。
- [ ] 建 `scripts/check`,把 lint/type/test/smoke 的最小集合收进去。
- [ ] issue 模板强制 `Source IDs / Context pack / Acceptance / Verification / Out of scope`。
- [ ] 每次只让 agent 做一个 ready feature。
- [ ] 写代码 agent 和 review/eval agent 分开。
- [ ] milestone 后做 product review,检查 non-goals 漂移和用户路径。

---

## 11. 推荐 prompt

### 11.1 生成 Source IDs

```text
Read docs/product-brief.md and draft docs/requirements.md, docs/flows.md, docs/data-model.md, docs/ui-contracts.md.

For every stable product fact, assign a Source ID:
- REQ-* for product requirements
- FLOW-* for user flows
- DATA-* for data model and invariants
- UI-* for UI states and interactions
- NFR-* for non-functional requirements

Keep IDs sparse. Only create IDs that future issues/tests/PRs should reference.
Do not write code.
```

### 11.2 生成 feature list

```text
Create feature_list.json from docs/.

Each feature must:
- represent one user-visible behavior
- cite source_ids
- have priority P0/P1/P2
- have status blocked/ready/done
- include a verification command or human-review rubric

Flag any unclear behavior instead of inventing it.
```

### 11.3 执行单个 feature

```text
Implement FEAT-...

Read:
- AGENTS.md
- feature_list.json entry for FEAT-...
- docs sections for its source_ids
- relevant existing code

Do:
- smallest working diff
- no out-of-scope features
- update tests/checks only as needed

Done when:
- acceptance criteria pass
- scripts/check passes
- verification passes
- progress.md records what changed and what remains
```

### 11.4 独立 review

```text
Review the current diff against FEAT-... only.

Check:
- Does it satisfy every source_id?
- Did it implement anything out of scope?
- Are error/empty/loading states covered?
- Does verification actually prove the behavior?
- Is there any simpler change that would satisfy the same contract?

Return findings by severity. Do not rewrite code unless asked.
```

---

## 12. 参考文献（已核对｜截至 2026-07-02）

### OpenAI / Codex

**OpenAI Codex · Best practices**: <https://developers.openai.com/codex/learn/best-practices>
- 本文采用的 `Goal / Context / Constraints / Done when` 任务结构来自这里。
- 关键启发:复杂任务先 plan 或让 Codex 采访你;稳定项目指导放 `AGENTS.md`;不要只让 Codex 写代码,还要让它测试、检查和 review。

**OpenAI Codex · Prompting**: <https://developers.openai.com/codex/prompting>
- 关键启发:Codex 在 prompt、工具调用、文件读取、编辑、验证的 loop 中工作;小任务更容易测试和 review;长任务需要明确 goal 和完成标准。

**OpenAI Codex · Custom instructions with AGENTS.md**: <https://developers.openai.com/codex/guides/agents-md>
- 关键启发:`AGENTS.md` 是 repo 级持久指导,适合记录目录、命令、约束、done definition。越靠近当前目录的指导优先级越高。

**OpenAI Codex · Agent Skills**: <https://developers.openai.com/codex/skills>
- 关键启发:可复用工作流应该沉淀成 skill;skills 用 progressive disclosure 管理上下文,只有触发时才读完整说明。本文的 task harness 将来可以升级成 repo skill。

**OpenAI Codex · Model Context Protocol**: <https://developers.openai.com/codex/mcp>
- 关键启发:MCP 用来连接外部文档、Figma、浏览器、GitHub、Linear 等上下文和工具。活数据不要复制进 prompt,让 agent 通过受控工具读取。

**OpenAI · A Practical Guide to Building Agents**: <https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf>
- 关键启发:agent = model + tools + instructions;先用强模型建立 baseline 和 evals,再优化成本;先把单 agent loop 做扎实,必要时再引入多 agent;guardrails 要分层。

### Anthropic / Claude

**Anthropic · Effective harnesses for long-running agents**: <https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents>
- 本文最直接的来源之一。
- 关键启发:高层 prompt 不能支撑长时间造 app;需要 initializer、feature list、progress log、增量 coding agent、干净状态和结构化交接。

**Anthropic · Harness design for long-running application development**: <https://www.anthropic.com/engineering/harness-design-long-running-apps>
- 关键启发:长任务 harness 不只是拆任务,还要让 agent 持续验证 app 是否真的工作,尤其要补浏览器自动化和 evaluator loop。

**Anthropic · Effective context engineering for AI agents**: <https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents>
- 关键启发:context 是有限资源;问题不是塞更多 token,而是为每次推理选择最小高信号上下文。本文的 Source ID + Context Pack 正是这个原则的项目管理落地。

**Anthropic · Building Effective AI Agents**: <https://www.anthropic.com/research/building-effective-agents>
- 关键启发:先用简单 workflow,只有简单方案不够时才增加 agentic 复杂度;workflow 与 agent 要区分;tool / ACI 设计要像 API 一样认真。

**Claude Code Docs · Best practices for Claude Code**: <https://code.claude.com/docs/en/best-practices>
- 关键启发:先探索和计划,再实现;用 subagents 做研究和 review;新鲜 context 对代码审查更有价值。

**Claude Code Docs · How Claude remembers your project**: <https://code.claude.com/docs/en/memory>
- 关键启发:`CLAUDE.md` 是 Claude Code 的项目记忆入口,适合记录常用命令、风格、工作流和项目约定。

**Claude Code Docs · Common workflows**: <https://code.claude.com/docs/en/common-workflows>
- 关键启发:探索、计划、编码、测试、提交可以成为稳定工作流;长任务要善用 session / resume / branch 风格的上下文管理。

---

## 13. 一句话总结

> **从 0 vibe coding 的关键,不是把 PRD 写到更长,而是把 PRD 编译成 agent 每次能执行的一小段合约。** Source ID 解决"依据哪里",Context Pack 解决"读什么",Acceptance 解决"做什么",Verification 解决"怎么算完"。这四件凑齐,AI 才不是在替你想产品,而是在替你执行产品判断。
