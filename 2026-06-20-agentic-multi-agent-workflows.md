# Agentic 工作流与多 agent 并行开发

> 副标题:并行 agent 的真正瓶颈不是「能开几个」,而是**协调 + 集成 + 评审**。最佳实践在优化这三件,而不是把 agent 数量拉满。**分解质量 > agent 数量。**
>
> 日期:2026-06-20 ｜ 来源:evidata 仓库一次关于「单人开发者如何用多 agent 并行处理 issue backlog」的讨论,叠加当天亲历的一次工作流硬化实战(rulesets / auto-merge / 合并队列 / Codex 评审闭环)。
>
> 配套阅读:同目录 [2026-06-18-harness-engineering.md](2026-06-18-harness-engineering.md)(agentic coding 的工程外壳;本篇是它在「多 agent 协作」维度的延伸)。

---

## 0. 一句话定位

> **把 agent 当「跑得飞快的下属」,你当 tech lead。** 写代码的成本趋近于零之后,瓶颈上移到三件事:**决定做什么 / 评审 / 把多份改动合到一起**。所有多 agent 最佳实践,都是在压缩这三件的成本——**靠隔离(worktree)+ 编排(单一写者)+ 通过产物沟通(blackboard)**,而不是靠 agent 之间互相协商。

---

## 1. 去掉营销滤镜:实验室里到底怎么干

主流不是「一群自主 agent 在共享 backlog 上抢活」,而是 **human-in-the-loop 的编排,在边界清楚的独立任务上逐步加并行**。几条贯穿原则:

1. **人始终是架构师和集成者(integrator)。** agent 负责产出,人负责「做什么 + 合到一起 + 守质量」。
2. **Plan → execute → verify,小步快跑。** 先出方案(plan mode)→ 人确认 → 执行;PR 尽量小;强 CI + 强评审兜底。
3. **并行靠隔离,不靠协商。** worktree 是物理隔离;真正难的是**逻辑隔离**——让并行任务不碰同一批文件。
4. **「全自主 agent 集群无人值守造功能」目前更多是 demo,不是日常。** 可靠的日常是编排 + 评审。

> 记住:**决定吞吐的是你把活切得多干净,不是你开了几个窗口。**

---

## 2. 核心认知:并行的集成成本是「超线性」的

两个 agent 改了相邻的代码,省下的写码时间会被 **merge 冲突 + 返工 + 评审** 吃光。所以:

- 并行收益 ≈ 任务独立性 × 分解质量 −(集成成本 + 评审成本)
- 当任务**文件/包不相交**时,集成成本接近 0,并行才真正提速。
- 当任务**耦合**(改同一个 schema / port / 公共类型)时,**别并行**——串行做,或塞进同一个 session。

---

## 3. 并行三难,逐个给机制

### 3.1 去重:多个 agent 不选中同一个任务

两条路,**强烈推荐第一条**:

- **编排者发牌(orchestrator / dispatcher)——根本上消灭竞争。**
  一个「lead」session 是 board 的**唯一写者**:它挑任务、派给 worker、负责合并。worker 是**无状态执行器**,只做被指派的那一个 issue。没人去抢,自然不重。
  → Claude Code 里有现成原语:**Workflow 工具**(确定性 fan-out 出 N 个 subagent,自带并发上限、worktree 隔离、结果汇总;subagent 只向父级汇报)。「一个有界任务拆成多份并行」直接用它,别手搓协调。

- **租约 / 认领(claim / lease)——去中心化时的并发控制。**
  每个 agent 开工**第一步必须原子认领**:`gh issue edit <n> --add-assignee @me` + 打 `status:in-progress` 标签(或在 Project 里把卡片移到 In-progress 列)。约定:只挑「未指派 + 未 blocked + 优先级最高」。GitHub assignee 不严格原子,但 1 人 + 2~4 agent 的体量撞车概率极低。

### 3.2 依赖序:按依赖顺序进行

**把依赖显式编码,让「未解锁的任务根本不进 Ready」**:

- 用 GitHub 原生 **sub-issues / issue dependencies**(blocked by / blocking)当依赖图;或轻量约定:正文写 `Depends on #N` + 未满足时打 `blocked` 标签。
- **状态机**:一个 issue 只有当依赖全部 **merged** 时,才从 `Blocked` → `Ready`。编排者只派 Ready 的。
- **合并也按依赖序串行**:strict 必需检查要求分支 up-to-date,所以「**并行写、串行合**」是常态;合并队列一个个过,后面的 rebase 到前面结果上。

### 3.3 通信:agent 之间会不会沟通

**默认不直接对话,而是「通过产物(artifacts)沟通」——blackboard 模式**:

- agent 间自由聊天 = 非确定、易跑偏、难复现,**避免**。
- 通信总线 = **编排者** + **共享产物**(issues / PRs / `main` / `docs/tech-spec`)。
- 谁改了共享契约,就**先合进 `main`**,其它 agent rebase 后自然看到。
- Claude Code 里:subagent 只向**父 session** 回报,彼此不通;父 session 就是总线。跨 session 有 SendMessage / session 管理,但小体量下**让 lead(你或一个 lead agent)当总线**最简单。
- 铁律:**真正要互相等结果的任务别并行。**

---

## 4. 推荐拓扑(单人开发者)

```
            ┌─────────────────────────────────────────────┐
            │  Backlog —— GitHub Issues + 一个 Project 板    │
            │  Ready → Blocked(deps) → In-progress → ...     │   ②依赖门:只派 Ready
            └───────────────────────┬─────────────────────┘
                                    │
                          ┌─────────▼──────────┐
                          │   Lead / 编排者      │  单一写者:grooming·派活·集成
                          └─┬────────┬────────┬─┘
                  ①认领=assign+label │        │            (lease,去重)
                    ┌───────▼─┐ ┌───▼────┐ ┌─▼──────┐
                    │worker 1 │ │worker 2│ │worker 3│   各自 worktree·1 issue·1 PR
                    └───────┬─┘ └───┬────┘ └─┬──────┘
                            └───────┼────────┘
                          ┌─────────▼──────────┐
                          │  串行合并队列         │  rebase-merge,按依赖序
                          └─────────┬──────────┘
                          ┌─────────▼──────────┐
                          │       main          │  CI + Codex + 线程 resolve 门控
                          └─────────┬──────────┘
                                    └──→ main 推进,各 worker rebase(③靠产物同步)
```

要点:
1. **GitHub Project(免费)当 board**(见 §5)。
2. **worktree-per-agent,一对一对一**:`git worktree add ../wt-<issue> -b <type>/<slug>-<issue> origin/main`。**永远别两个 agent 共用一棵工作树。**
3. **认领协议**:开工先 assign self + `status:in-progress`。
4. **依赖**:sub-issues 或 `Depends on #N` + `blocked`;Ready = 依赖全 merged。
5. **分解时就切成文件/包不相交**(`apps/web` vs `packages/db` vs `packages/core/*`),issue 里标 `Touches: <dir>`——这是并行能否真提速的**命门**。
6. **串行合并队列**:rebase-merge、按依赖序、auto-merge + `allow_update_branch` 自动追新。
7. **lead 角色**:你,或专跑一个「只 grooming + 派活 + 集成、不写代码」的 session;有界大任务交给 **Workflow** 编排。

> 现实甜区:**1 个 lead + 2~3 个 worker** 基本是单人开发者的上限。再多,评审/集成变成瓶颈,吞吐反降。

---

## 5. GitHub Project 是什么(我缺的那块「免费 Jira」)

**GitHub Projects(新版,Projects v2)= GitHub 内建的轻量规划/追踪工具**,定位类似 Jira / Linear / Trello,但长在 issues、PR 之上、且**免费**。

- **层级**:账户级 / 组织级(**可跨多个仓库**),不是旧的「每仓库看板」(classic Projects,已弃)。
- **本质**:一个建在 Issues / PRs 之上的**灵活视图层**。同一批条目可切三种视图:
  - **Table**(电子表格)— 批量编辑、排序、分组。
  - **Board**(看板)— 按 `Status` 字段分列拖卡。
  - **Roadmap**(时间线)— 按日期/迭代排甘特图。
- **自定义字段**:`Status`(单选)、`Priority`、`Iteration`(冲刺)、文本 / 数字 / 日期 / 单选。**`Status` 就是你给 agent 用的状态机**(`Backlog / Ready / In-progress / In-review / Done`)。
- **内建自动化(built-in workflows)**:条目加入→设默认 Status;Issue/PR 关闭→`Done`;**PR 合并→`Done`**;**auto-add**(按 filter 自动把符合条件的新 issue 纳入);auto-archive。
- **依赖可视化**:sub-issues / issue dependencies 是 **issue 级**能力,Project 里能看父子与 blocked 关系。
- **Insights**:燃尽图 / 按字段统计。
- **入口**:你的 GitHub 头像 → Projects → New project → 选模板(Board / Table / Roadmap);或 repo 的 Projects tab 关联。

> 一句话:**Project = issues 的「看板 + 电子表格 + 自动化」外壳**。给它一个 `Status` 字段 + 几条自动化,你就有了一个能让多个 agent 读「Ready 列」、开工移「In-progress」、合并自动进「Done」的协作板——零成本替代 Jira。

---

## 6. 给 evidata 的落地 checklist

- [ ] 建一个 GitHub Project,`Status`:`Backlog / Ready / In-progress / In-review / Done`,加自动化(PR 开→In-review,合→Done,新 issue auto-add)。
- [ ] worktree-per-agent;AGENTS.md 写明「一个 agent / 一棵工作树 / 一个 issue」。
- [ ] 认领协议入 AGENTS.md:开工先 assign self + `status:in-progress`,只挑 Ready+未指派+优先级最高。
- [ ] 依赖:启用 sub-issues 或 `Depends on #N` + `blocked`;Ready = 依赖全 merged。
- [ ] 分解时切文件/包不相交,issue 加 `Touches (paths)` 栏。
- [ ] 串行合并队列:rebase-merge + auto-merge + `allow_update_branch`(已开)。
- [ ] 有界大任务用 Workflow 工具编排,而非手搓多 session。

---

## 7. 反模式(踩了就疼)

- ❌ 两个 agent 共用一棵工作树 → 互相覆盖未提交改动。**必须 worktree 隔离。**
- ❌ 并行处理耦合任务(同改 schema/port)→ merge 地狱。**耦合就串行。**
- ❌ 让 agent 之间自由聊天协调 → 非确定、难复现。**走编排者 + 产物。**
- ❌ 一把梭把 N 个 PR 同时合 → strict 检查下后面全 `BEHIND`,还是得一个个 rebase。**串行合。**
- ❌ 盲挂 auto-merge 到实质代码 PR → 可能赶在 Codex 评审前就合。**docs/CI 可以,代码 PR 等评审。**

---

## 8. 今日实战印证

当天做仓库工作流硬化,真切验证了上面的机制:

- **worktree 隔离救场**:主工作树正有另一个 agent 的未提交 WIP;所有 docs/CI/license PR 全在独立 worktree 里做,主树原封未动。
- **串行合并的 `BEHIND` 级联**:CodeQL PR 先合,推进 main,导致另外两个 PR 立刻 `BEHIND`;靠 `allow_update_branch` + `gh pr update-branch --rebase` 逐个追新才合。
- **评审闭环门控真的拦住了问题**:给 `protect-main` ruleset 开了 `required_review_thread_resolution` 后,一个 PR 被 Codex 的 **P2** 卡成 `BLOCKED`——它精准指出「新加的 `Part of #N` 约定」与「旧的『每个 PR 都写 `Closes #N`』」自相矛盾(partial PR 一合会把 umbrella 误关)。**这正是「线程不 resolve 不让合」的价值:最高危的 fixup 时刻不再无人复审。** 修正 → 👍 → resolve → 自动合。

---

## 9. 工具映射速查

| 需求 | Claude Code | GitHub |
|---|---|---|
| 隔离并行 | worktree / `isolation:"worktree"` | — |
| 编排 fan-out | **Workflow 工具**(并发上限/汇总) | — |
| 子任务汇报 | Agent 工具(subagent→父级) | — |
| 后台长任务 | `run_in_background` | Actions |
| backlog / 看板 | — | **Issues + Projects** |
| 依赖 | — | sub-issues / dependencies |
| 认领/去重 | — | assignee + label |
| 质量门控 | — | rulesets(必需检查 + 线程 resolve) |
| 串行合并 | — | rebase merge + auto-merge + `allow_update_branch` |

---

## 10. 一句话总结

> **多 agent 不是「开更多窗口」,是「把一个 tech lead 的工作流程显式化」**:把活切干净(文件不相交)、用 board + 认领去重、用依赖图定序、用编排者 + 产物代替闲聊、用串行合并队列 + 强门控守质量。**先把分解和集成做扎实,再谈加 agent。**
