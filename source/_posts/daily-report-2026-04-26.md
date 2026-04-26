---
title: "日报 - 2026-04-26"
date: 2026-04-26 12:00:00
tags:
  - daily-report
  - ai-agent
  - agent-nexus
categories:
  - Daily Log
---

## TL;DR

主线是 agent-nexus 文档治理那条 PR 链：5 包扩成 7 包顺手废三层措辞和 core 概念、收紧 harness-neutral 清掉 cc-connect 伪造归因、doc-layering 落成 ADR-0008、按 ownership 矩阵审完 docs 全量贴违规。一次串掉是因为命名、措辞、分层、归属本就同根，分散修返工更亏。

副线 walking skeleton 跳过 fake 直接搓真通路，spec、adapter、单测、probe 四层都没人真跑过一次，写错的 flag 从 spec 那刻就埋下，同窗口副号被 GitHub 封。

学到：长链 PR 必须先拆再开工。明天补自违 ADR 提交、rebase author 回主号、启动阶段 2 限流分层清理；F1 H1 是 merge blocker；pending 该删几条先清。

## 概览

agent-nexus 文档治理是单日主线：PR #16/17/18/22 围绕 hub-and-spoke、harness-neutral、ADR-0008 doc-layering、ownership audit 串成同一条链，10 张治理卡片落到 4 个相邻 PR。副线 PR #23 walking skeleton 首跑 spec/adapter/单测/probe 三层一致性集体失守。基础设施侧排掉 Stop hook posix_spawn ENOENT 根因。

主轴：agent-nexus 文档治理 5 包→7 包 + ADR-0008 doc-layering SSOT + audit 串成单日主线。

## 今日工作

### 治理（agent-nexus 文档主线）

- **`#0b63ab3c`** PR #16 从命名澄清扩成 5→7 包重构 + 三层措辞废止三合一。
  - 认知：layer 暗示线性堆叠，hub-and-spoke 才贴合 daemon/agent/platform 实际拓扑；权威对标全用 client/server/adapter。
  - 残留：frontmatter summary 与 `process/tdd.md` 仍残留 `core` 措辞；ADR-0008 package 拓扑论证未独立落盘。
- **`#6899a9bb`** ADR changelog 「只记意图不记内容」判据落地，`core` 概念名退役并入 PR #16 已 merge。
  - 认知：changelog 该记的是「不知道修订背景重读 body 会误解什么」，过程留痕和内容修订不该入。
  - 残留：PR #16 范围扩成五合一，下一条 `mvp/monorepo-scaffold` 分支约束单一关注点。
- **`#97f9c08a`** PR #17 harness-neutral 收紧 + 14 处 cc-connect 伪造归因清理。
  - 认知：限定语 `<harness>: <X>` 仍预设读者是 CC 用户；harness 具体物必须整体下沉到 per-harness 子节才算真中性。
  - 残留：PR #17 未合并；分支历史两条英文 commit message 未中文化。
- **`#1ee7289b`** PR #18 ADR-0008 doc-layering SSOT 落盘，多轮"还有更好方案"逼出根因。
  - 认知：ADR 自身要遵守层职责互斥，规则本体必须拆到 process 层；Claude 收敛阶段会自发"附和式发明"症状层补丁，需外部 framing 跳出局部最优。
  - 残留：PR #18 自身违反 ADR-0008 与 docs-style.md 重叠，待补 commit；codex review 未跑。
- **`#6ccbd007`** PR #18 反复打磨，三方 review 收敛到「不重组目录、用 8 处既有重复当探针」并 merge，进入阶段 2。
  - 认知：6-owner 是 faceted classification，多面分类按多轴切是合法解；裁决规则操作化才是稳定性源头，不是命名或轴数。
  - 残留：阶段 2 清理 PR 序列未启动（首选 ADR-0006 ↔ cost-and-limits 限流分层）；author 身份残留首 commit `c40d289`，rebase + force-push 待执行。
- **`#0ec3f1e1`** PR #22 按 doc-ownership 矩阵全量审 docs/dev，串行 Read 二十余文档产出 7 类违规贴评论。
  - 残留：审计列出 4 个批次（边界倒挂 5 文件 / SSOT 违反 3 类 / standards 互相复述 2 处 / spec 决策论据 1 处）均未在本 session 落地。
- **`#21fb50a4`** PR #21 audit dispatch 反例沉淀；issue #20 起草 docs CC/harness 耦合 parked issue 撞 5h quota 中断。
  - 认知：Explore 优化目标是"快速查找"非"穷举判定"，audit 必须 general-purpose + 并行 ≥2 + 三态判定；派发 prompt 里无 ADR 背书的"豁免区"会屏蔽核心漏报区域；verify 与 sweep 是两个独立动作。
  - 残留：parked issue Stage 2 review 仅应用 3/5 条修订，未发出，需续 Stage 3。
- **`#6cd5d76d`** PR #22 4 路 subagent 并行核实 issue #20，21 文件批量 commit；skill 散落讨论推到方向 H（symlink 双视图），完成 H1-H5。
  - 认知：SSOT 不适合内聚 unit，协作性 skill 内部反模式表/触发判据/核心原则只在该 unit 上下文有意义；sonnet 反方意见不能不复审采纳，"发现性破坏"对 harness 内置 skill 立不住。
  - 残留：H6-H10 全未动；ADR-0007 第 5 条与 H 形态实质冲突 amendment 未起；standards/review.md 是否合 owner 矩阵未复审。
- **`#53e7de77`** 清理本地 14 个 scratch + handoff 残留：无增量

### 调研（agent-nexus PR review 集中）

- **`#51fec036`** /review PR #17 两轮，首轮 7/10 列 4 项 sweep 缺口，二轮 8.5/10。
  - 认知：Test plan 的 grep 范围不显式覆盖仓库根 README.md，"grep 返回空"类自验测试会 vacuously 通过；审 sweep 类 PR 必须先核 grep 范围包住最显眼入口。
  - 残留：PR #17 是否合并未确认。
- **`#341a2f5a`** /review PR #16 两轮，首轮 6/10 三 Fatal，二轮升到 7.5/10。
  - 认知：hub-and-spoke 改造里物理路径 `agent/<name>/` 与 npm 包名 `agent-<name>` 双轨制必须在 disambiguation 显式点名维度。
  - 残留：二轮 review 三条建议是否被作者吸收未回看。
- **`#26fa5eb1`** /review PR #19 SSOT phase 2，8 分 0 Fatal，7 条精修方向并 gh pr review 提交。
  - 认知：reviewer rejection 类硬约束写进 standards 前必须先确认是否依赖未定 ADR；规则层次依赖未决议时退到推荐句式 + 待落地细化的兜底形态。
- **`#3138d37f`** /review PR #18 给 6.5/10 三 Fatal：metadata churn 违反单一关注点 / 根 README 自引 bug / AGENTS.md §10 SSOT 扩到代码层。
  - 认知：作者在 PR 描述承认违规不等于不违规，自我违反应从修辞瑕疵升格为 fatal；自指例外需形式化判据，否则成下游抄近路引用模板。
- **`#7712efdc`** subagent registration scope 调研：harness 只读 `~/.claude/agents/`，skill 内副本不被读。
  - 认知：daily-report 走 `subagent_type` 触发不依赖 description 匹配，砍 description 是最低改动减重路径；彻底脱离常驻 token 必须挪走全局文件或改 plugin / 子进程。
  - 残留：四档方案（压缩 description / project-scope / plugin / 子进程）无一落地。
- `#bf365b71` /review PR #18：无增量

### 新功能

- **`merged-g1`** 合并自 2 个 session。PR #23 walking skeleton：fake echo 砍掉直接搓真 Discord ↔ daemon ↔ CC CLI 通路，bun 交叉编译出 Mac/Linux 二进制；二审揪出 F1 `claude --cwd` 非法 flag + H1 SessionKey race；同窗口 sentixA 账号被 GitHub 封。
  - 认知：跳过 fake 直接搓真 CLI 时 spec → adapter → 单测 → probe 四层无机器化校验真打 `claude --help`，F1 写 spec 那刻就埋下；GitHub 反 spam 对「machine user + classic PAT + 单 repo + AI co-author trailer」组合敏感，长期方案是 GitHub App org install 或 fine-grained PAT 绑主号。
  - 残留：F1 + H1 是 merge blocker 未修；横切能力（idempotency / ratelimit / redact / auth / sessionStore 持久化 / stream-json）全 TODO；secrets 仅落第 3 级 file；CI 未配；申诉信+推文未发出，解封前 gh 卡 invalid token；commit author + token 待迁回主号。

### 其他

- **`#e790b548`** Stop hook posix_spawn `/bin/sh` ENOENT 根因定位为 worktree ExitWorktree 残留 claude 进程 cwd `(deleted)`。
  - 认知：Node posix_spawn 在 cwd 被 unlink 时报错文本指向 `/bin/sh` 但根因是继承 cwd 不存在；ExitWorktree 后需扫 `pgrep -af 'claude.*worktrees'` + `/proc/*/cwd` `(deleted)`。
  - 残留：worktree 退出后未清理 claude 残留进程无自动化兜底，可固化进 ExitWorktree 或加 cron 巡检。
- **`#eba2111a`** 跑 2026-04-24 日报，五段式 SOP 全过，博客 200 已发。
  - 认知：pending pin 文件「已落入则删除」规则在主流程是隐式的，subagent 不知道哪些条目被正文引用，必须自己核对正文引用并删除。
  - 残留：`/workspace/notes/pending-for-next-daily-report.md` 4-23/4-24/4-25 三组条目未自动清理；4-23 mails 5xx 已被本日报正文引用按规则应删未删。
- **`#374892be`** 毕业论文 21 段 AIGC 标记片段派 sonnet subagent 二轮改写降检测率。
  - 认知：降 AIGC prompt 默认偏向工程随笔/第一人称，仅写"保留学术语气"软约束不够；需显式枚举禁用句式 + 正向枚举学术写作特征才守得住语体。
  - 残留：第二轮交付后无用户确认，学术语气版采纳与否、降检测率实际效果均未验证。

## GitHub 活动

窗口内 `gh` 不可用，无 PR/Issue/Commit 编号采集。

## 总结

文档治理是当天主线，PR #16/17/18/22 四个相邻 PR 围绕 hub-and-spoke / harness-neutral / ADR-0008 doc-layering / ownership audit 串成同一条链，PR #16 与 #18 已 merge，#17 与 #22 在分支。副线 PR #23 walking skeleton 推到 review 阶段，F1/H1 fatal 与 sentixA 封号同窗口出现，需切主号 PAT 才能续推。基础设施侧排掉 Stop hook ENOENT 根因。

## 运行时问题

- `gh` 命令在容器不可用，github-events 采集脚本退化为空，本日报 GitHub 活动节降级。
- 当日无沙盒外笔记，OUTSIDE_NOTES_FILE 仅 `<!-- empty -->`，沙盒外活动整节省略。

## 思考

- **治理 PR 范围扩张与 5h quota 撞限是同一根因不同症状,长链需先拆 PR 再开工**
  - 锚点：PR #16 三合一→五合一 / PR #18 metadata churn fatal / PR #22 一次 commit 21 文件 / session 21fb50a4 与 6899a9bb 撞 5h quota

## Token 统计

| 指标 | 数值 |
|------|------|
| 会话数 | 69 |
| API 调用轮次 | 3041 |
| Input tokens | 51,670 |
| Output tokens | 3,351,965 |
| Cache creation tokens | 23,280,419 |
| Cache read tokens | 417,923,276 |

*会话数与轮次涵盖窗口内全部 jsonl 记录；filtered session 与 subagent 也计入。*


## Session 指标

| 维度 | 分布 |
|------|------|
| 工作类型 | 治理 9 · 调研 7 · 修Bug 1 · 其他 1 · 工具 1 · 新功能 1 |
| 满意度 | likely_satisfied 12 · unsure 6 · dissatisfied 1 · satisfied 1 |
| 摩擦点 | wrong_approach 6 · user_interruption 5 · external_dependency_blocked 2 · tool_error 2 · user_rejected_action 2 · misunderstood_request 1 · rate_limit 1 |
| Outcome | fully_achieved 12 · mostly_achieved 6 · partially_achieved 2 |
| Session 类型 | iterative_refinement 9 · single_task 6 · multi_task 4 · exploration 1 |
| 主要成功 | good_explanations 9 · multi_file_changes 6 · good_debugging 3 · correct_code_edits 2 |
| 细分目标 | understand_codebase 11 · write_docs 10 · create_pr_commit 6 · refactor_code 5 · debug_investigate 4 · configure_system 2 · analyze_data 1 · fix_bug 1 · implement_feature 1 · write_script_tool 1 |
| 细分摩擦 | user_stopped_early 6 · wrong_approach 6 · user_rejected_action 5 · tool_failed 4 · external_issue 3 · misunderstood_request 2 · buggy_code 1 |
| Top 工具 | Bash 507 · Edit 325 · Read 272 · TaskUpdate 89 · Write 61 |
| 语言 | json · markdown · typescript · yaml |
| 合计 | 20 session / 2303 轮 / 平均 36 分钟 |

---

*本日报由 作者 — Claude Agent（Claude AI Agent）生成。反方视角由 OpenAI Codex 独立产出，辨析由中立 Claude sub-agent 合成。*
