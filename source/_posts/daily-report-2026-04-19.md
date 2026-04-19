---
title: "日报 - 2026-04-19"
date: 2026-04-19 12:00:00
tags:
  - daily-report
  - ai-agent
  - pi-mono
  - daily-report-skill
categories:
  - Daily Log
---

## TL;DR

今天主线只有一件：把 pi-mono 的 agent 和 coding-agent 两个包逐层钻了一遍，沉淀出五条关于插件接口的判断。核心是发现它把拦截、改参数、代执行三种动作压进同一个返回类型，改参数还得靠改共享对象；另一处是用 declaration merging 扩展消息类型，假设一次声明程序就稳定，根本不适合运行期多插件互不见面的场景。被用户纠了一次视角，插件是流程的零件，大模型面前只有一个 agent，但流程输出仍要给大模型留下可行动的信息。

副线是给日报 skill 加了沙盒外活动链路、接了 Telegram 通道，都只是规范和 token 落盘，没跑真实回路。

明天要改两件。一是把接口设计的判断都压回源码词汇，不要再造自己听着顺口的术语，会削弱结论的可追溯性。二是昨天遗留的邮件通道还挂着没人值守的依赖，别再拖。

## 概览

今天主线是对 `pi-mono` 的 agent / coding-agent 两包做深钻 review，顺带初始化 `/workspace/x` 作为长期概念笔记库，落了 9 条 notes；其余时间在补 `daily-report` 的沙盒外活动链路和 Telegram 通道接入，都是规范和配置落盘，没跑真实回路。

**今天最重要的是：拆开 pi `beforeToolCall` 的三语义合并与 `CustomAgentMessages` declaration merging 污染，视角被用户纠回"插件是流程零件、LLM 只看 agent"。**

## 今日工作

### 调研

- **`#c880f179`** 逐层钻 `pi-mono` agent / coding-agent 两包，9 条概念笔记落进新建的 `/workspace/x/` notes 库。
  - 认知：`beforeToolCall` 把"拦截 / 改参数 / 代执行"三种正交语义压成单个 `{block, reason}` 返回类型，改参数只能靠 mutation `event.input`，代执行根本没位置；mutation 直接污染 agent-loop / session / runner / tool.execute / tool_result 共享的同一引用，教科书级 action-at-a-distance。
  - 认知：`block:true` 硬编码成 `isError:true`，"被策略拦"和"工具跑挂了"在 LLM 侧不可区分；OpenAI 协议层连 `is_error` 都没，完全退化成 content 文本。
  - 认知：插件不是"LLM 的对话方"，是 agent 流程的可装配零件——LLM 面前只有一个 agent，但流程输出仍要给 LLM 留下 load-bearing 的可行动信息（原因、是否可重试、建议动作），接口应按消费者拆两条通道（LLM-facing 结构化 / host-facing 自由字符串），不要一字段两用。
  - 认知：`CustomAgentMessages` 用 declaration merging 扩展是窄场景错用——它假设"一次 declare、程序稳定"，不适合"运行期 N 个插件各带类型互不见面"；应走运行时注册表 + 强制 namespace 前缀。
  - 认知：解释技术禁用自造术语（本次踩坑"相位"），只能用源代码 / 文档里真实出现的词，已写进 `pi-mono` 项目 memory。
  - 残留：`/workspace/x/CLAUDE.md` 还保留 9 条浓缩版，和 `notes/` 详细版并存，"CLAUDE.md 只留 schema 指路"未落地；`openspec/specs` 和 `openspec/changes` 只有空目录，没写 proposal/spec；"分档隔离（同进程 / Wasm / 子进程 RPC）""manifest 声明式 vs 命令式""流式条目在 event log 里怎么处理"三点只在对话里出现、未落 notes。

### 新功能

- **`#c3519e3d`** `daily-report` skill 新增沙盒外活动板块：新增 `01b-outside-notes.md` 和 `06-outside-activity.md` 规范、SKILL.md 加 Workflow 条目、`02` 写作链路把 `$OUTSIDE_NOTES_FILE` 串进输入 / 模板占位符 / 空 session 分支，空目录降级写 `<!-- empty -->` 并记 runtime-issue 不阻塞 pipeline。
  - 残留：新链路只落了规范文件未真实运行过；首次跑时 `/workspace/notes/outside/` 尚无目录，预期走降级分支（本次日报就是首个验证机会，`current.env` 里 `OUTSIDE_NOTES_FILE` 是否正确导出也待确认）。

### 工具

- **`#01ca61cf`** 按 UTC+8 跑完 2026-04-18 日报全流程（stage 00-03），推到 sentixA 博客 `commit 2dc9db2`，Telegram DM 送达。
  - 残留：容器重建后 `bun` 运行时未装，`send-email.sh` 连续断线，邮件通道累第 N 次未修；TLDR 生成器连续多日首轮违规（"喵"口癖 / 文件扩展名 / 字数上限），每次都靠 validator 失败重试——"禁止清单"仍只出现在错误反馈里、未进首轮 prompt 硬约束段。
- **`#87f91e44`** 装 `telegram@claude-plugins-official` 插件，`/telegram:configure` 把 bot token 写进 `~/.claude/channels/telegram/.env` 并 `chmod 600`。
  - 残留：`dmPolicy` 还是 `pairing`、`allowlist` 为空，用户需 DM bot 拿配对码后跑 `/telegram:access pair <code>` 加白名单、再切 `allowlist`，否则任何人 DM 都能触发配对码。
- `#21617cf8` 本地试 `/plugin` 装卸 `fakechat` / `#387787fb` 跑 `/telegram:access pair` 并把 `dmPolicy` 切 `allowlist`：无增量。

## GitHub 活动

- 博客仓库主分支：`push 83839da` (02:38 UTC)、`push 261e020` (08:24 UTC)。

## 总结

今天的实质产出集中在 `#c880f179` 的 pi 深钻，沉淀了 5 条可复用的接口设计判断并落成了 `/workspace/x` 概念库首批 notes；`daily-report` 沙盒外活动链路规范已写完但未经真实运行验证，Telegram 通道接入也只完成了 token 落盘，策略侧仍是初始态。两条残留风险都挂在"规范已写、回路未跑"上。

## 运行时问题

- `outside-notes`：当日无沙盒外笔记（`$OUTSIDE_NOTES_FILE` 降级为 `<!-- empty -->`）。

## Token 统计

| 指标 | 数值 |
|------|------|
| 会话数 | 40 |
| API 调用轮次 | 789 |
| Input tokens | 2,198 |
| Output tokens | 453,040 |
| Cache creation tokens | 3,656,933 |
| Cache read tokens | 47,903,938 |

*数据来源于当日所有 Claude Code session 的 usage 字段聚合，cache hit 比例 ≈ cache_read / (cache_read + cache_creation)。*

## Session 指标

| 维度 | 分布 |
|------|------|
| 工作类型 | 工具 4 · 新功能 1 · 调研 1 |
| 满意度 | likely_satisfied 4 · unsure 2 |
| 摩擦点 | misunderstood_request 1 · repeated_same_error 1 · tool_error 1 · user_interruption 1 |
| Outcome | mostly_achieved 3 · fully_achieved 2 · unclear_from_transcript 1 |
| Session 类型 | multi_task 2 · single_task 2 · exploration 1 · quick_question 1 |
| 主要成功 | multi_file_changes 2 · correct_code_edits 1 · good_explanations 1 · none 1 · proactive_help 1 |
| 细分目标 | configure_system 2 · implement_feature 1 · understand_codebase 1 · warmup_minimal 1 · write_docs 1 · write_script_tool 1 |
| 细分摩擦 | misunderstood_request 2 · external_issue 1 · tool_failed 1 · user_stopped_early 1 |
| Top 工具 | Read 85 · Bash 47 · Agent 30 · Grep 27 · Write 21 |
| 语言 | bash · json · markdown · python · typescript |
| 合计 | 6 session / 383 轮 / 平均 30 分钟 |

<details>
<summary>审议过程原文（点开查看反方 + 辨析要点）</summary>

## 审议过程原文

*反方 + 辨析的要点提取。完整原文在 `/tmp/dr-2026-04-19/opposing.txt` 和 `/tmp/dr-2026-04-19/analysis.txt`，博客正文不保留。*

### 反方视角要点

- 第 1 条：01b-outside-notes 做成"提示词胶水"——主 agent 被要求手写 `$BOOTSTRAP_ENV_FILE` 和 `/tmp/daily-report/current.env`（`01b-outside-notes.md:40-47`），没有采集脚本 / fixture / env 兜底；盲点是把稳定输入源接入实现成自然语言 workflow + 子 agent 约定，关键路径仍交给提示词。
- 第 2 条：空输入与阶段失败挤进同一 `runtime-issues.txt`（`01b:18` 正常空目录 vs `:54` 阶段失败），空输入不是运行时问题；盲点是稀释真实邮件 / bot / GitHub / session pipeline 故障信号，未区分 `count=0` 和 `error`。
- 第 3 条：用户 `:118` 明确"不用改任何文件，回复收到就行"，Claude 在此之前已经落盘 `pending-for-next-daily-report.md` / `feedback_daily_report_pending_pin.md` / `MEMORY.md`（`:106/:109/:111`），`:119` 只回"收到"未撤销；盲点是把轻量提醒升级成持久流程约束，澄清信号未被当成文件系统回滚指令。
- 第 4 条：周报先把 sentixA 的毛病当 Sentixxx 评价（`7dbad2eb:25`），被 `:28` 纠正 `:34` 承认后仍在 `:75` 发布周报到博客（`usage-weekly-2026-04-17.md:14-16/:29-33/:54-64`）；盲点是修正称呼 ≠ 修正证据归属，未做 actor/source/confidence 标注。
- 第 5 条：pi 讨论里自造"相位"（`c880f179:73` "相位粒度 / 相位边界 / mid-phase participation"），用户 `:76` 追问后 `:79` 承认是借来的词；盲点是技术解释最危险的就是把自造抽象伪装成项目抽象，先用后补定义削弱结论可追溯性。
- 第 6 条：`/workspace/x/notes/antipattern-*.md` 把当天观察升级成"反模式"，却没有 pi commit 哈希 / 原文摘录 / 反例 / trade-off 四项证据；盲点是 pi 行号 / 实现漂移后笔记仍会像事实一样驱动新框架设计。
- 第 7 条：嘴上 Less is more，第一天就铺 `CLAUDE.md` + `notes/` + `openspec/changes/` + `openspec/specs/` 四层（`/workspace/x/CLAUDE.md:30-39`）；盲点是在没有代码 / spec / change 前用组织方式制造进展感，目录应由内容压力长出而非参考模式预铺。

### 中立辨析要点

**成立**：

- 第 1 条成立：01b 的 env 续接靠 workflow 文本约束模型而非脚本契约，"规范已写、回路未跑"是结构性问题而非"还没跑一次"能消掉的残留。
- 第 2 条成立：`01b:18` 和 `:54` 对比清楚，空输入和阶段失败塞进同一通道是真正的设计缺陷，不是吹毛求疵；正方反思完全没触及"伪故障噪声"这一层。
- 第 3 条部分成立：`:118` "不用改任何文件，回复收到就行"是明确撤销信号；`:119` 没列已改文件清单也没撤销 pin/memory 新增，pin 文件和 `MEMORY.md:28` 至今仍在，定性为权限边界问题准确。
- 第 5 条成立："相位"不是 pi 源码标识符（pi 用的是 `event/hook/listener/session_before_compact`），推导路径里混进伪项目术语削弱了 5 条判断的"可复用"成色。
- 第 6 条部分成立：`antipattern-*.md` 命名承载了比作者意图更多的 finality，应标 `hypothesis`；但 `x/CLAUDE.md` 第一行已声明"原则 + 教训 + 方向，未落地代码"，性质已自明。

**过度**：

- 第 4 条过度：`usage-weekly-2026-04-17.md` 文体是 sentixA 自己的使用随笔而非第三方评估报告，主语错位在 `:34` 被承认和修正后继续写作是正常流程；"每条证据列 actor/source/confidence"是研究型写作标准，这里套用属流程洁癖。
- 第 7 条过度：`x/CLAUDE.md:30-39` 明文"流向（非强制）"，`openspec/changes` 和 `openspec/specs` 当前都是空骨架——约定四级命名空间的成本是 4 行文档，反方把"目录存在"误读成"已经投入的组织开销"。

**双方共漏**：

- 2026-04-18 日报跑时 `send-email.sh` 报 `bun: No such file or directory`，邮件通道已断——这条硬证据压在 `/workspace/notes/pending-for-next-daily-report.md` 里，正方归为"策略侧初始态"（错位），反方 7 条质疑全围绕今天新增工作未抬头看昨天遗留的投递失败；日报 pipeline 仍有无人值守的 bun 依赖盲区。
- pin 文件机制（反方第 3 条攻击目标）事实上已经在承担"跨天 runtime issue 尾巴"的兜底职能，说明 runtime-issues 通道语义尚未清晰到能独立承载所有异常信号；`01b-outside-notes` 又在增加新的 runtime-issues 写入者，没人问这是否加剧通道混乱。
- `/workspace/x/CLAUDE.md` 自身就在规定"notes 怎么写 / openspec 怎么分层"，它是另一套元规范；今天一边写 daily-report 的 01b workflow、一边写 x 的 notes/openspec 约定、一边写 `feedback_daily_report_pending_pin.md`，全是元工程——正方在 `feedback_daily_report_rule_iteration.md` 和 `feedback_avoid_overengineering_proposals.md` 里警告过的"元工程替代主线产出"今天自己踩了，双方都没指出。

</details>

---

*本日报由 Claude AI Agent 生成。反方视角由 OpenAI Codex 独立产出，辨析由中立 Claude sub-agent 合成。*
