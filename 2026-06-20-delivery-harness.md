# 交付外壳:把"确定性外壳"延伸到开发流程本身

> 副标题:[Harness Engineering](2026-06-18-harness-engineering.md) 讲的是**产品**外壳(模型周围那圈确定性);本篇讲**交付**外壳——开发与发布流程自己的那圈确定性。核心论点:**AI 把写代码变便宜,于是把错误注入也变便宜了——所以验证和门控必须更硬、更自动,把"对不对"的反馈压到几分钟。**
>
> 日期:2026-06-20 ｜ 来源:一次给单人/小团队仓库做"工作流硬化"的实战,全部配置可复用。
>
> 配套阅读:[Harness Engineering](2026-06-18-harness-engineering.md)(产品外壳总论)、[多 agent 工作流](2026-06-20-agentic-multi-agent-workflows.md)(并行协调)、[判断力与品味](2026-06-20-taste-and-judgment.md)(为什么 verification 是延迟的解药)。
>
> 适用对象:用 AI 高频改代码、想要"快但不塌"的人。这是**工程实践层**——可以照着配。

---

## 0. 一句话定位

> **AI proposes, the pipeline disposes.** 把你的判断**编译成自动门控**(constraints as code),让"错的事进不了 main"不靠自律、靠流水线。AI 写得越快,这层刹车越不能省。

---

## 1. 核心原则:为 AI 的速度配刹车

一个因果链值得刻在脑子里:

> AI 让"写代码"成本 → 0 ⟹ "注入一个看似合理、其实错的改动"成本也 → 0 ⟹ **唯一能跟上的是:让"验证对不对"的成本也 → 0(快、自动、强制)。**

所以交付外壳的每一项,本质都在回答同一个问题:**怎么让一个错误在几分钟内、自动地、被挡在 main 之外?**

---

## 2. 分支与合并纪律

- **永不碰 main**:用 ruleset(新版,非 classic protection)保护默认分支——`deletion` + `non_fast_forward` + `required_linear_history`。
- **rebase-only 线性历史**:仓库设置只留 rebase(关 squash / merge commit)**且** ruleset `allowed_merge_methods:[rebase]`——两处一致,不自相矛盾。
- **必需检查 strict**:`strict_required_status_checks_policy: true`(分支必须 up-to-date 才能合)。配 rebase 是教科书组合。
- **并行写、串行合**:strict 检查下,前一个 PR 合并会让其余 PR 变 `BEHIND`,**必须逐个 rebase 再合**——这是常态,不是 bug。用 `auto-merge` + `allow_update_branch` 让 GitHub 自动追新分支,把这件事自动化掉。

> 实战印证:一次连合 6 个 PR,每合一个,其余立刻 BEHIND;靠 `gh pr update-branch --rebase` 逐个追新才合掉。**别想着一把梭同时合 N 个。**

---

## 3. 验证门控(verification shell)

把"对不对"压成一个 PR 上几分钟就出结果的必需检查:

- **一个必需 job 里塞满**:typecheck + lint + format-check + 单元测试 + 真实依赖集成测试(如真 Postgres service)+ E2E smoke。
- **关键手法:把 lint/format 放进**必需 job**,而不是旁路检查**——很多团队把 lint 设成 non-required,于是它形同虚设。门控的一部分 = 真的门控。
- **加 SAST**:CodeQL(公开仓库免费),PR + push + 周度,`security-extended`。对"只读 SQL / 不泄露"这类安全主张的产品尤其相关。先设 advisory,成熟了再提为必需。

---

## 4. 评审闭环:AI 评审是一层,人是裁决

- **AI 自动评审**(如 Codex):每个 PR 留 P0/P1/P2 行内评论。读它、修 P1/P2(P0 必修)、👍 致谢、re-push。
- **强制线程 resolve**:ruleset 开 `required_review_thread_resolution`。**这条堵住了最危险的洞——"按评审改完的那次 fixup,恰恰是最没被复审的"。** 开了它,每条 AI 评论必须显式 resolve 才能合,而不是"👍 一下就合过去"。
- **分工**:AI 评审负责广度铺开(找),**人负责裁决**(这条 P2 是真问题还是误报)——呼应[决策纪律](2026-06-20-deciding-with-ai.md)的"裁决 vs 点头"。

> 实战印证:开了 thread-resolution 后,一个 docs PR 被 Codex 的 **P2** 卡成 `BLOCKED`,它精准指出"新加的 `Part of #N` 约定"和"旧的'每个 PR 都写 `Closes #N`'"自相矛盾。修正 → 👍 → resolve → 自动合。**门控真的拦住了一个会自相矛盾的约定。**

---

## 5. 供应链 / 密钥硬化(公开仓库尤其)

一个处理密钥的公开仓库,最高危、最常见的事故就是"一把 key 被 push 上去"。全部免费、几分钟开:

- **secret scanning + push protection**:在 commit 落地前直接拦下密钥。
- **Dependabot**:alerts + security updates(+ 可选 version updates)。
- **私有漏洞上报** + **SECURITY.md**:给漏洞一个私下报告的入口,别让人开公开 issue。
- **action 钉 SHA**:`uses: actions/checkout@<40位sha> # v4`,别用可变 tag(`@v4` 能被 tag 劫持,cf. tj-actions/changed-files 2025);配 `dependabot.yml` 的 `github-actions` 生态自动升 SHA。

---

## 6. 工作树隔离(配合多 agent)

- **worktree-per-agent,一对一对一**:`git worktree add ../wt-<issue> -b <branch> origin/main`。
- **永不两个 agent 共用一棵工作树**——未提交的 WIP 会互相覆盖。
- 细节见[多 agent 工作流](2026-06-20-agentic-multi-agent-workflows.md)。

---

## 7. PR 作为契约

- **PR 描述精简、面向实现**:意图 + 改了什么 + 怎么验证 + 验收 + `Closes #N`。AI 评审读 diff + PR body(不读链接的 issue),所以 body 要自足;但**别把设计灌进 PR**(设计在 issue / spec 里)。
- **收尾才 `Closes #N`,partial 用 `Part of #N`**——否则一个分阶段 PR 会把 umbrella issue 误关。
- **design-before-code**:带 `needs-design` 的 issue,先在 issue/spec 里和人定好设计,再写代码。设计评审发生在设计阶段,不在实现 PR 里。

---

## 8. 落地清单(照着配一个新仓库)

- [ ] ruleset 保护 main:禁删 / 禁强推 / 线性历史 / 必需检查 strict / 只 rebase。
- [ ] 一个必需 CI job:typecheck + lint + format + 测试(含集成)+ smoke,全在门内。
- [ ] CodeQL(advisory 起步)。
- [ ] AI 评审 + `required_review_thread_resolution`。
- [ ] secret scanning + push protection + Dependabot + SECURITY.md + private reporting。
- [ ] action 钉 SHA + dependabot(github-actions)。
- [ ] auto-merge + allow_update_branch(让串行合并队列自动追新)。
- [ ] PR/issue 模板:精简面向实现 + design-before-code + Closes/Part of 约定。
- [ ] 多 agent 时:worktree 一对一,并行写串行合。

---

## 9. 一句话总结

> **交付外壳 = 把你的判断编译成自动门控,让 AI 的高速度不会变成高熵。** 它和产品外壳是同一套哲学(AI 提议、确定性的壳来校验/记录/守住),只是作用对象从"模型输出"换成了"你和 AI 的每一次提交"。**速度越快,刹车越要硬——而最好的刹车是自动的、几分钟就响的。**
