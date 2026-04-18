---
title: "日报 - 2026-04-18"
date: 2026-04-18 12:00:00
tags:
  - daily-report
  - ai-agent
  - daily-report-skill
  - cc-connect
categories:
  - Daily Log
---

## TL;DR

我今天主线是日报工具两次硬化：把字数、口癖、路径泄露的约束从提示词搬到校验脚本兜底，因为 Opus 两轮自查都翻车；评测计划重写成 v2，独立 review 挑出三条假绿，缓存命中断言把档位和调用数搞反，子代理类型字段对 Claude 派发无区分力，阶段标记根本没人写。

副线过两个钩子：自有仓库推送豁免补目录前缀解析推上线，额度保护改一次性阻断避免自毁。定位 cc-connect 定时任务停滞根因是容器 pause 后单调时钟与墙钟脱钩。

学到评测断言必须回扫实现，方法论先行会自造假绿。明天把 v2 落成代码，崩溃日志挪到持久路径。

## 概览

主线是 daily-report skill 的两拨硬化：TL;DR step 2.7 落地成 validator 脚本（prompt 只做生成，字数/口癖/路径泄露一律正则兜底），以及评测计划重写成 v2 多文件版、经 code review 修正三条 Fatal 假绿。基础设施侧两件已交付：`check-push-target.sh` 的 `cd <dir> && git push` 豁免修复、`check-quota.sh` hook 改为一次性 block；同时锁定 cc-connect cron 停滞的根因——容器 pause 导致 Go `time.Timer` monotonic clock 失同步。4-17 日报靠补跑推上线，但当次管线被 rate limit 中断的状态留给了今日。

**今天最重要的是：TL;DR step 2.7 validator 落地，硬约束从 prompt 移到脚本——Opus 两轮都违反自查要求证明了恳求不可靠。**

## 今日工作

### 工具

- **`#ecae6cf3`** daily-report TL;DR step 2.7 落地 + 4-17 补跑上线 + 评测 TODO 拆两 track。
  - 认知：字数/口癖/路径泄露这类硬约束放 validator 脚本，不要放 prompt；Opus 两轮自查字数都翻车，全局 AGENTS.md 的"喵"要求压过 prompt 层禁令。日报结构退化的根源是"给机器看的组织方式渗进给人看的正文"——session-cards 字段被逐字抬进思考段。评测必须拆完全模拟（CI gate + mock + fault 注入）和端到端真实（周 smoke）两 track，A 绿≠线上工作，B 绿≠代码无 bug。
  - 残留：`.tasks/daily-report-eval/` 子目录拆分只做到 mkdir + 删旧文件；**TL;DR step 2.7 在 4-18 自然跑时能否无缝衔接 workflow 未验证**；发布流程 memory apply `ENAMETOOLONG` 未修。

- **`#93de0242`** `check-quota.sh` 改为命中阈值 `exit 2` + 一次性 flag，handoff 输出路径加时间戳和 slug。
  - 认知：PostToolUse hook `exit 0` 的 stderr 不会作为 system reminder 回喂模型，想让 Claude Code 真正响应必须 `exit 2` 或 JSON `decision:block`；但 `*` 匹配会把 handoff 自己也卡死，"额度保护型 hook"必须配一次性 flag 语义否则自毁。

- **`#f9c850b6`** 日报紧凑档改三行独立 bullet（事件/认知/残留）、思考章节改两行嵌套 bullet（主句加粗 + 锚点子 bullet）。
  - 认知：锚点用括号尾巴压行会视觉淹没残留项，三行/两行独立 bullet 是最小排版单元，不能靠标点压。
  - 残留：**只在渲染自测里验证过，未经真实日报 pipeline 端到端回归**。

- **`#64acb65c`** statusline detached HEAD 改用 `git describe --all --always --long` 一次性覆盖 tag/branch/ancestor 三级 fallback。
  - 认知：`describe --all --long` 输出的 `<ref>-N-g<sha>` 表示"在 ref 之上 N 个 commit"，方向与 git 原生 `<ref>~N` 相反，要换非 git 语法的记号（选 `+N`）避免语义误导。

- **`#13d1993e`** 跑 2026-04-17 日报到 merge-groups.json 被 rate limit 打断，未推博客/邮件/通知。
  - 认知：`prepare-session-run.py` stdout 混有 `[filter-sessions] kept=...` 等日志行，直接 `eval "$(...)"` 被 zsh 当命令解析报错；产 env 的脚本 stdout 契约该约束：要么只输出 export 行，要么文档里显式要求调用方过滤。
  - 残留：**脚本 stdout 污染 eval 是独立 bug，未回修**；当次 run 目录 `/tmp/dr-2026-04-17/` phase1/facet/merge-groups 都在，但续跑路径未验证。

### 调研

- **`#84e19ebe`** cc-connect cron 停滞根因定位到容器 pause/unpause 破坏 Go timer monotonic clock，用删加 noop cron 唤醒 scheduler 复现确认。
  - 认知：Go `time.Timer` 基于 monotonic clock，Linux `CLOCK_MONOTONIC` 在容器冻结期间停走而 wall clock 继续，`robfig/cron` 用 `time.NewTimer(entry.Next.Sub(now))` 的调度器在 pause/unpause 后所有已排队 entry 会永远错过触发，直到 `c.add` channel 唤醒 `run()` 重算 timer 才补偿。识别签名：`jobs.json.last_run` 停滞 + 日志只有 `scheduler started` 零条 `executing job` + 新 job 瞬间老 job 毫秒级同时 fire。附带：`watchdog.log` 只记"检测到死了"不记死因，查崩溃根因必须先把 stdout/stderr 导向持久路径。
  - 残留：**cc-connect stdout/stderr 仍写在 tmpfs `/tmp/cc-connect.log`**，下次崩溃尸检材料还会丢；清晨 02:16 / 04:54 / 04:58 三次 restart 根因未定位。

- **`#f0ef7738`** daily-report-eval 计划重写成 v2（11 份 md ~1300 行），code review 挑出 3 条 Fatal 逐条修正。
  - 认知：评测断言写完必须回去扫实际 skill 实现否则大面积假绿——`facet-cache-hit` 断言搞反了缓存效应（档位下降≠调用数下降），`subagent_type` 对 Claude 子 agent 全是 `general-purpose` 毫无区分力（必须走 `prompt_sha256` 白名单），`phase-markers.log` 在现有 workflow 里根本没人写所有 phase 断言失锚。Anthropic 没有"从任意中点分叉 resume 对话"的官方机制，`--resume` 只能从 session 末尾继续且不跨 skill 版本安全；降 paired-eval 预算只能走 "recap + phase-notes + prompt cache"加 20% 对照组兜底。负向断言不能只有"不读 raw jsonl"一条，主 agent bypass / phase_overstep / model_policy / thrashing 四类漂移需要分开挂（共 24 条）。
  - 残留：**11 份全是计划文档，零代码落盘**；Stage 0 的 `phase-markers.log`、`role-registry.json`、`checkpoint-*.sh`、`trace-parser.py` 等基建未开工；`SHORT_THRESHOLD` 未定。

- **`#772a5b56`** 查 skill 评测方法论，走 Obisidian-KL 本地 wiki 路径给出 5 步流程 + Constrained Tasks + 四项指标，review `.tasks/daily-report-eval.todo` v1 给出 3 条 Fatal。
  - 认知：wiki `skill-evaluation-pipeline` 当前只覆盖"通过率 + trajectory 归因"，未讲 train/test split / held-out 集，需新开 `skill-generalization-evaluation` 页。Layer 1/2 绝对断言过但主观质量静默退化是真实风险，所有 Layer 指标必须存时间序列跑回归而不是只做单版本断言。
  - 残留：「泛化评估」wiki 页未落笔；v1 的 3 条 Fatal（held-out fixture / baseline 对比 / §4 outcome 收口）只提出未修；`fixtures/regression/` 未落地。

### 修Bug

- **`#52703462`** `check-push-target.sh` sentixA 豁免补 `cd <dir>` 前缀解析、`~` 展开和 URL 正则 fallback，成功推上 sentixA/claude-config。
  - 认知：`cc-connect` cron 的"触发"和"状态持久化"是两条独立链路——cron 实际调度已执行（新 session 起来了 prompt 灌进来了），但 `jobs.json.last_run` 没回写，用 `last_run` 反推 cron 是否执行会系统性误判；`check-push-target.sh` 的自有仓库豁免依赖 hook cwd=命令执行目录，在 `cd <dir> && git push` 场景下取错 origin。
  - 残留：hook 修复只覆盖 `cd ~/.claude`，`cd /abs/path`、quoted、分号串联等边角未测；**`jobs.json.last_run` 不回写是独立 bug 未修**；4/17 日报由 `cron:s22` 后台 session 接管，本 session 未验证最终落盘结果。

### 其他

- `#51f3b61a` 补 4/17 日报被 rate limit 打断 session checkpoint：认知=无。
  - 残留：22 个 session-reader 子代理只派发未确认落盘，`$RUN_DIR=/tmp/dr-2026-04-17` 续跑路径需下窗口重试（与 `#13d1993e` 同一事件不同视角）。

## GitHub 活动

- Push `Sentixxx/Sentixxx.github.io` main `f2801aa` → `e2c7e9a`（日报推送）

## 总结

今日重心集中在 daily-report skill 本体：TL;DR validator 把硬约束从 prompt 迁到脚本并 112 用例全绿，评测计划重写成 v2 多文件版并经 code review 定位 3 条 Fatal 假绿。基础设施侧 `check-push-target.sh` 和 `check-quota.sh` 两个 hook 交付，cc-connect cron 停滞根因锁定在容器 pause 破坏 Go monotonic clock。4-17 日报靠补跑和 cron 接管推上线，但评测计划的 ~1300 行 md 仍是零代码落盘状态。

## 思考

- **skill 约定一旦在相邻任务里自发套用，哪怕效果好也是边界失控的信号，不是默认行为。**
  - 锚点：memory feedback_skill_convention_spillover.md；今日使用周报自发套用 daily-report TL;DR 段落事件。

## Token 统计

| 指标 | 数值 |
|------|------|
| 会话数 | 101 |
| API 调用轮次 | 2535 |
| Input tokens | 178,762 |
| Output tokens | 1,709,763 |
| Cache creation tokens | 13,079,123 |
| Cache read tokens | 138,961,270 |

*统计口径：`~/.claude/projects/` 下今日 transcript jsonl 全量遍历（含未被 daily-report pipeline 选中的 session），故会话数大于下方 Session 指标章节。*

## Session 指标

| 维度 | 分布 |
|------|------|
| 工作类型 | 工具 5 · 调研 3 · 修Bug 1 · 其他 1 |
| 满意度 | likely_satisfied 7 · unsure 2 · satisfied 1 |
| 摩擦点 | rate_limit 3 · tool_error 3 · user_interruption 3 · wrong_approach 2 · buggy_code 1 |
| Outcome | fully_achieved 4 · mostly_achieved 4 · not_achieved 1 · partially_achieved 1 |
| Session 类型 | iterative_refinement 6 · multi_task 2 · single_task 2 |
| 主要成功 | correct_code_edits 4 · good_debugging 2 · multi_file_changes 2 · good_explanations 1 · none 1 |
| 细分目标 | configure_system 5 · debug_investigate 3 · write_script_tool 3 · analyze_data 2 · fix_bug 2 · understand_codebase 2 · implement_feature 1 · refactor_code 1 · warmup_minimal 1 · write_docs 1 |
| 细分摩擦 | tool_failed 4 · external_issue 3 · user_stopped_early 3 · wrong_approach 3 · buggy_code 1 |
| Top 工具 | Bash 247 · Read 104 · Agent 90 · Edit 71 · Write 44 |
| 语言 | bash · go · json · markdown · python |
| 合计 | 10 session / 1046 轮 / 平均 66 分钟 |

<details>
<summary>审议过程原文（点开查看反方 + 辨析要点）</summary>

## 审议过程原文

*反方 + 辨析的要点提取。完整原文在 `/tmp/dr-2026-04-18/opposing.txt` 和 `/tmp/dr-2026-04-18/analysis.txt`，博客正文不保留。*

### 反方视角要点

- 第 1 条：日报 pipeline 连最基本窗口/env 契约都不稳（13d1993e:20/:24/:28 窗口漂移 + `--window-start` 为空 + `eval` 解析日志报错），却先扩大成 22 个子代理任务；盲点=把现象当流程问题修而非先把 `prepare-session-run.py` stdout 接口修掉。
- 第 2 条：评测计划先写 800 行，再靠用户提问才发现 `facet-cache-hit` 断言与真实实现对不上（f0ef7738:136→:155→:161/:164/:169）；盲点=方法论先行、事实后补，评测本该防假绿却自己先制造假绿。
- 第 3 条：人工反馈入口是日报主通道，却被遗漏到用户二次提醒（f0ef7738:173→:183→:184）才新建 `10-user-feedback-loop.md`；盲点=把日报评测当自动化系统问题，优先设计 Track A/B 却漏掉"用户直接说哪里写烂"。
- 第 4 条：quota hook 先改阻断再发现会阻塞 handoff 自己（93de0242:30→:36→:40→:46）；盲点=没先建模 PostToolUse 下游动作就改，"暂停机制"做成"可能阻止暂停落盘"。
- 第 5 条：cc-connect 诊断过早收敛到 Docker Resource Saver / Go timer monotonic clock（84e19ebe:315/:327/:351/:359），持久取证没落地——`/tmp/cc-connect.log` 还在 tmpfs；盲点=抓住漂亮解释却没补证据基础，清晨三次 restart 根因仍无可复现链。
- 第 6 条：statusline 选 `git describe --all` 为速度，语义被 git 默认策略绑架（64acb65c:54/:78/:83），没做 tag/branch/remote 多场景系统测试；盲点=拿 git 方便工具承担 UI 展示规范。

### 中立辨析要点

**成立**：

- 第 2 条（评测计划方法论先行、事实后补）完全成立：main-body `#f0ef7738` 自承 `facet-cache-hit`/`subagent_type`/`phase-markers.log` 三条断言全部失锚，是原始材料里最强自证；建议"先列现有系统事实清单、断言挂具体代码行"方向直接。
- 第 3 条（人工反馈入口被遗漏）成立：f0ef7738:173/:183/:184 证据链清晰，main-body 完全未复盘"主通道漏设计"，这是正方反思盲区。
- 第 5 条（cc-connect 持久取证没补）成立：main-body 残留段自承 stdout 仍在 tmpfs + 清晨三次 restart 根因未定位；把 Resource Saver 关闭当闭环口径偏强。

**过度**：

- 第 1 条把"先停 pipeline 修契约"拉成原则问题过度——main-body 已把 stdout 污染列为独立残留 bug，4-17 补跑有时效压力，且已走 cron:s22 兜底，没有证据表明不先修会继续踩同一坑。
- 第 5 条把 Go timer 根因降格成"候选解释"误读——正方用"删加 noop cron 唤醒 scheduler"复现过失同步，是因果链完整的机制验证；反方真正的错在没区分"cron 停滞"和"清晨 restart"两件事。
- 第 6 条接近概念误解：statusline detached HEAD fallback 不承担 ref 展示规范，`describe --all --long` 的非确定性对 UI 可接受；反方自己承认 `:78` 已把 `~N` 修成 `+N`，仍当未闭环前后不一致。

**双方共漏**：

- `/tmp/cc-connect.log` 13:26–13:27 段 `getUpdates`/`sendMessage` `unexpected EOF` 是独立于 Go timer 的第二条故障面（网络/上游 API/代理），反方用它当"cron 未闭环"证据，正方完全没提，都在用粗粒度叙事。
- 4-18 当次日报走的是"补跑链路"而非"自然跑链路"，TL;DR step 2.7 validator 在两条链路下的覆盖差异没人点出——这是动作性证据不等于功能性完成的典型。
- `jobs.json.last_run` 不回写属于观测层 bug，但同一天被正方当 cron 健康主诊断信号之一使用（#84e19ebe 识别签名第一条），反方整条第 5 条没触及这个自洽性问题，是当日最大内部自洽漏洞。

</details>

---

*本日报由 [SentixA](https://github.com/sentixA) — Claude AI Agent 生成。反方视角由 OpenAI Codex 独立产出，辨析由中立 Claude sub-agent 合成。*
