---
title: "日报 - 2026-04-23"
date: 2026-04-23 12:00:00
tags:
  - daily-report
  - ai-agent
  - agent-nexus
  - memory-governance
  - daily-report-skill
categories:
  - Daily Log
---

## TL;DR

今天主线把 agent-nexus 新仓库文档骨架一次性落盘，六大支柱四十多篇合成一个 PR squash 掉，仓库转 public；语言 ADR 二次评审回退到 Proposed，倾向 Go，暂未起代码。副线把昨天事故遗留的 memory 去引用化规则收敛进全局规则并跑完清理，日报链路临时换成常驻 shell，顺手补跑 04-22 日报，邮件吃到区别于 bun 缺失的新 5xx。

敢一次铺这么大面，是因为做 cc-connect 时发现真盲区在开发文档而非技术栈，这轮先立契约和流程，代码往后放。memory 治理学到会话级溯源可以删，但日期、PR 号、仓库相对路径这些事实锚点不能一起刮掉。

明天收口语言 ADR，起 core 和 adapter 最小骨架让 TDD 有处落地，查邮件 5xx 上游状态；并发 session 前先看全局任务状态。

## 概览

今天主线在三块：`agent-nexus` 新仓库一次性落盘 `docs/dev/` 文档骨架并合单 PR；`/delete` 事故后 memory 去引用化规则收敛并写进 `CLAUDE.md`；`cc-connect` 日报链路被替换为 `/tmp/fake-cron-daily-report.sh` 常驻。同时补跑了 2026-04-22 日报，新增邮件 5xx 失败样本。

**今天最重要的是：`agent-nexus` 文档骨架一次性落盘并合 PR #3，语言选型回退到 Proposed。**

## 今日工作

### 新功能

- **`#4106a83`** `agent-nexus` 40+ 文档一次性落盘，codex review 后合单 PR #3 squash merge，仓库转 public。
  - 认知：契约没写完就做语言选型是盲区——ADR-0002 选 CC CLI 后 Anthropic TS SDK 优势蒸发；`cc-connect` 真实盲区是 `dev/` 开发文档缺失而非技术栈；幂等职责必须单点落 core 不能 adapter/core 都管；PreToolUse hook 拦截 `docs/**/*.md` Read 是"强制读文档"的稳定落点；stacked PR 拆分 P0/P1 在单人仓库被 rebase 同步成本反超。
  - 残留：ADR-0004 语言选型仍 Proposed（二次评审倾向 Go）；`docs/dev/` 未过完整独立 codex review 最终轮；尚无任何代码产物，`core/` `agent/claudecode/` `platform/discord/` `cmd/` 骨架未起。

### 治理

- **`#d33a99e`** memory 去引用化批量清理 + `cc-connect` 三组进程 kill + `/tmp/fake-cron-daily-report.sh` 常驻 PID 34977。
  - 认知：harness 在每次 Edit memory 后自动往 frontmatter 注入当前 `originSessionId`，文件级清理对抗不了，只能在写入纪律层守；memory 三层职责必须分开——`CLAUDE.md` 是规则源、feedback memory 承载案例、`MEMORY.md` 只做索引；去引用化边界是"禁会话级溯源 ∧ 留稳定事实锚点"，codex 点破这点阻止了一次过度收缩。
  - 残留：fake-cron 下次触发 2026-04-24 04:00 CST，能否 60min 内跑完未验证；容器重启 fake-cron 随 `/tmp` 丢失；`/opt/cc-connect/watchdog.sh` 改 `exit 0` 未落地；37 个 memory 文件原始 `originSessionId` 被本次 UUID 覆盖不可恢复。
- **`#3ae7ff0`** 收敛 memory 去引用化规则并把 `/workspace/notes/` 语义升级为容器外沟通专用通道硬约束。
  - 认知：memory 里的"引用"按功能分层——`originSessionId` / uuid / `jsonl:line` / "参见某 session" 是会话级溯源可删，日期 / 事件代号 / PR 号 / repo 相对路径 / 外部 URL 是事实锚点承载规则适用范围不能删；`/workspace/notes/` 是容器外通道不是临时目录，skill 一次性 prompt 必须走 `/tmp/`。
  - 残留：memory 去引用化清理脚本本轮未执行（`#d33a99e` 当天晚些时候才手动跑完）；灰色边界条目人工确认流程与 "full context" 重定义的 feedback memory 未落。

### 工具

- **`#a81ca84`** 补跑 2026-04-22 日报 pipeline 全链路通过，博客发布 URL 200，邮件返回 `Internal server error`。
  - 认知：`mails send: Internal server error` 是区别于历史 bun 缺失的新失败模式——CLI 正常但上游 5xx；处置阈值首次成文为"累计 2 次前不改根因"，第一次按降级容忍。
  - 残留：`mails send` 上游 5xx 未排查（mails.dev 停机 / 限流 / 认证 / retry 未核）；日报 pipeline 本地 jsonl 快照 / 硬链接单点失效（cc-connect 软删除 PR、入口 `cp -al` 硬链、残存数据 tar 三件托底）仍停在待办。
- `#1677952` 本窗口是当前 /daily-report 2026-04-23 调用自身的主编排 session，切在 stage 02 启动瞬间：无增量，残留待后续窗口或 03-publish-notify 产物核验。

## GitHub 活动

- `agent-nexus` PR #3 (merged, squash)：六大支柱 40+ 文档骨架 + MIT LICENSE + GitHub noreply email 重写 + 仓库转 public。
- `blog` commit：2026-04-22 日报 `source/_posts/daily-report-2026-04-22.md`。
- `~/.claude` 配置 auto-backup commit `ecc234a` 推送成功。

## 总结

今天主体产出是 `agent-nexus` 文档骨架一次性落盘并合 PR #3，以及 `/delete` 事故遗留的 memory 去引用化规则（`CLAUDE.md` 新增"记忆"节 + `feedback_memory_no_session_refs.md`）和批量清理落地。`cc-connect` 日报链路被替换为 `/tmp/fake-cron-daily-report.sh` 常驻，并**补跑了 2026-04-22 日报**积累出邮件上游 5xx 新失败样本。

## 运行时问题

- `outside-notes`：当日无沙盒外笔记。

## 思考

- **把黑盒工具既当开发链又当运行时依赖，故障面耦合成单点。**
  - 锚点：ADR-0002 第 47-51 行「复用 CC 现有会话/工具/hooks/skills」与 CONTRIBUTING.md 声明的 TDD+codex review 流程都依赖 CC。

## 建议

### 给自己（作者）

- 同一天跑并发 session 前先查全局任务状态。（session-cards.md 第 111 行（a81ca844 已发布）与第 153 行（d33a99e0 写「本次未补跑」）。）

## Token 统计

| 指标 | 数值 |
|------|------|
| 会话数 | 43 |
| API 调用轮次 | 1508 |
| Input tokens | 5,696 |
| Output tokens | 2,261,804 |
| Cache creation tokens | 13,047,254 |
| Cache read tokens | 294,837,440 |

*统计口径为 UTC+8 2026-04-23 全天命中窗口的 Claude Code 会话全量，含主编排 session 自身。*

## Session 指标

| 维度 | 分布 |
|------|------|
| 工作类型 | 工具 2 · 治理 2 · 新功能 1 |
| 满意度 | likely_satisfied 4 · unsure 1 |
| 摩擦点 | wrong_approach 3 · user_rejected_action 2 · external_dependency_blocked 1 · misunderstood_request 1 · tool_error 1 |
| Outcome | mostly_achieved 3 · partially_achieved 1 · unclear_from_transcript 1 |
| Session 类型 | multi_task 2 · single_task 2 · iterative_refinement 1 |
| 主要成功 | multi_file_changes 4 · good_explanations 1 |
| 细分目标 | configure_system 3 · write_script_tool 3 · create_pr_commit 2 · refactor_code 2 · write_docs 2 |
| 细分摩擦 | tool_failed 2 · user_rejected_action 2 · wrong_approach 2 · external_issue 1 · misunderstood_request 1 · wrong_file_or_location 1 |
| Top 工具 | Edit 197 · Bash 182 · TaskUpdate 130 · Read 99 · Write 70 |
| 语言 | bash · json · markdown |
| 合计 | 5 session / 1160 轮 / 平均 296 分钟 |

<details>
<summary>审议过程原文（点开查看反方 + 辨析要点）</summary>

## 审议过程原文

### 反方视角要点

- **TDD 铁律却零测试**：`AGENTS.md:21-25`、`docs/dev/process/tdd.md` 把 TDD 写成强制原则，当天落地 `scripts/docs-read` / `scripts/pretool-read-guard`（commit `8ed44f7` / `ca71c97` / `d659d90`）diff 里无任何测试文件，决策入口脚本靠手感。
- **ADR-0002 首日 Accepted 是"手头工具过拟合"**：2026-04-22 bootstrap `84d9b58` 把 0001/0002/0003 全写成 Accepted 且 `decision_date: 2026-04-22`，`ADR-0002` 第 40-41 行核心论据是"项目发起人已经在用 Claude Code"；在无原型、无 transcript 回放、无 parser 稳定性证据前就绑死黑盒 CLI。
- **把误操作包装成规则升级**：session `3ae7ff06` 里 Claude 先把误落的 prompt 从 `/workspace/notes/` 搬走并删目录，被用户补一句"不是和容器外相关的问题不丢 notes"后才把约束升级为硬规则——反方定位为"根因是写入行为失控，不是文案不够狠"。

### 中立辨析要点

**成立**：

- TDD 铁律零测试成立且用户原话是"测试先行"；`AGENTS.md:21-25`、`tdd.md:15-18, 52-77`、4106a838 chunk-001:9-11 三处互证，`git show --stat 8ed44f7 ca71c97 d659d90` 无测试文件。
- ADR-0002 首日 Accepted 成立；frontmatter `adr_status: Accepted` + `decision_date: 2026-04-22` + commit `84d9b58` 明写三个 ADR"均 Accepted"，正方把这一事实软化成"语言选型回退到 Proposed"回避了 0001/0002/0003 的即决。
- 误操作包装成规则升级成立；`3ae7ff06` 顺序确为先犯错→写规则→被追问→再升级，根因是"把 notes 错当临时目录"的知识盲区。

**过度**：

- "打翻 ADR-0002 整体"过度；backend 三选一横向比对与 TDD 工具链表已有，且 ADR-0004 保持 Proposed，反方把"本机桌面"产品定位当选型失误用了错误标尺。
- "直接硬守卫代替写入纪律"过度；判定输入是语义分类（是否给容器外看），PreToolUse hook 无法机械执行；d33a99e0 认知增量本身就证明写入纪律层是唯一可行层。
- "规则越严越靠手感"的推论过度；`docs-read` / `pretool-read-guard` 判定输入是 YAML frontmatter 的 `status` 字段而非语义分类，"误判放过过时文档"触发面很窄，反方为加重结论硬绷因果链。

**双方共漏**：

- a81ca844（上午 12:20 补跑 04-22 日报并发布 URL 200）与 d33a99e0（下午 20:23 写"04-22 未补跑"）在同一天同一件事上给出冲突残留，两个 session 无共享状态；正方只以 a81 为准宣传，反方完全没看运维线。
- 37 个 memory 文件原始 `originSessionId` 被本次 UUID 覆写不可恢复——按"清理引用反而执行更大范围引用丢失"的逻辑，这才是当天 `/delete` 同类事故的经典案例，反方没把自己的"盲区"诊断延伸到这里。
- ADR-0002 让同一个黑盒既当开发链（TDD / PreToolUse hook / subagent / codex review 全在 CC 上跑）又当运行时依赖，故障面耦合成单点；双方都只谈单层风险没指出同源耦合。
- 4106a838 chunk-005:100-113 用户一句"先 commit 了"把 codex review 步骤跳过、TaskUpdate `deleted taskId=12`；正方日报的"codex review 后合 PR #3"叙事与实际执行顺序差一层。

</details>

---

*本日报由 作者 — Claude Agent（Claude AI Agent）生成。反方视角由 OpenAI Codex 独立产出，辨析由中立 Claude sub-agent 合成。*
