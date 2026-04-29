---
title: "日报 - 2026-04-28"
date: 2026-04-28 12:00:00
tags:
  - daily-report
  - ai-agent
  - agent-nexus
  - daily-report-skill
  - claude-config
categories:
  - Daily Log
---

## TL;DR

今天主要在 agent-nexus 上转。把 8 个 subagent 跑出的全仓 review 收回来再过一遍——工业标准下 487 行结论，按"自己一个人用"重写出 174 行个人版。顺手把 stream-json 主路径开成 epic，写 ADR-0012，被 codex 反方和用户从 UX 顶回来后，从 416 行瘦到 265 行，UI 决策全部转成独立 issue。

值得记一笔：review 得按使用场景重做一次，不然 P0 会被工业噪声占满；ADR 写到 emoji 文案就越界，那是 PRD 的活。副线 weekly-report 也踩到同款坑——subagent 当过滤层，主线程就拿不到原始信号。

明天要补：双视角 review 没定下次派发会僵尸化；engine 那两条隐患还没开 issue；GitHub 采集器是 no-op，得搬到主 agent 用 MCP 重写。

## 概览

10 个 session（含两组合并卡）覆盖三条主线：agent-nexus 全仓治理弧（access control 修复 → 全仓 review → P0 校准 → stream-json epic 化 + ADR-0012）、claude-config memory 减法整理与 weekly-report 流水线改造、4-27 日报续做。当日产出多在分支或 review 中，少量已合并；治理与认知沉淀比代码体量更显著。

**今天最重要的是：agent-nexus 全仓 review 的两次评级校准，以及为此改装的 stream-json epic + ADR-0012。**

## 今日工作

### 治理

- **`#merged-g2`** agent-nexus 治理弧合并自 5 个 session：access control 重命名 `allowedUserIds` 与 `parseInbound` tagged union 化（PR #51）、`requirement-clarification` 流程文档（PR #52）、8 subagent 全仓 review 收 487 行后另起 174 行 personal-use 版、自查推翻昨日 `error→Idle` 派生结论、P0 列表 triage 后开 epic issue #56 关联 #45/#28/#30/#54/#55。
  - 认知：review 评级必须随调用形态与威胁模型校准两次——工业 → 个人威胁、形式 → 现实可达；单做一次会留"工业噪声/不可达条目"占 P0。单点修与架构演进重叠时按 ETA 决定要不要做最小版，#54 让位 stream-json epic 避免双重劳动；access control 扩语义必须区分 control plane / data plane，「空 = fail-closed」在 admin 安全、在数据面最危险。
  - 残留：**PR #51 / #52 未合主线，需 Discord 手动验 `/reply-mode` + 非 allowlist ack 拒**；`testGuildId` bot 缺席仅 error 不阻断启动未集成测试覆盖；`dispatchImpl` 顶层 catch 吞 sendInput 错（engine.ts:226-235，真实 session 句柄泄漏点）未修；`buildSlices` UTF-16 切 emoji surrogate、多切片中途失败丢 sentIds 未修；protocol/spec 字段漂移（`SessionConfig` 多字段 / `AgentInput.attachments` / `AgentSession.state` 6 态 / `OutboundMessage.embeds` 等）未对齐；epic #56 ADR draft 此前未起（已由 `#770c801c` 补上 ADR-0012）；#53 / #55-A 两个正交小 PR 未启动；最后一句"要我接着干吗"未回复。
- **`#be35b9ed`** memory 系统按 04-22/04-23/04-24/04-27 四起召回失败做减法，A/B/C/D 4 subagent 并行合并主题簇 29 → 10，MEMORY.md 86 → 67 行，落 `/workspace/notes/memory-recall-meta-issues.md` 记录 3 条元问题。
  - 认知：memory 规则越多召回率越低，**减法是唯一有效动作而非再写一条更高优先级规则**；用户态规则、框架态 hook 注入、仓库可见性是三类不同解决域，规则适用对象不在仓库可见性内时规则等于不存在。
  - 残留：`/workspace/notes/memory-recall-meta-issues.md` 三条元问题待容器外/未来人工处理：`originSessionId` hook 改首次创建写入或重命名为 `lastModifiedSession`、`~/.claude/` 的 .gitignore 给规则类 skill 开白名单或迁出 `.claude/`、35 条仍超注意力窗口召回率监测无方案。
- **`#86b8d10a`** PR #52 claude[bot] review 4 条意见：在 `workflow.md` 主路径插 step 1.5、`requirement-clarification.md` 补 `## Acceptance` 与收紧触发条件第 3 条；commit 748774f 后已 squash merged。
  - 认知：仓库已有独立 worktree 时响应 PR review 应直接开新 worktree 操作 PR 分支，不要在主仓库做分支切换 + stash 来回倒，否则极易把两份 WIP 混进同一 stash 栈（本次误 pop feat 那份 stash 引发 3 文件 conflict）。
- **`#09f26e15`** `agent-nexus` 仓库根下 `.memsearch/` 目录归属选型：最初建议放 repo `.gitignore` 类比 `.gemini/.continue/.cursor/`，被用户纠正后改放 `~/.config/git/ignore`。
  - 认知：harness 产物的 gitignore 归属判断标准是「是否每个 collaborator 都会产生」而非「现有仓库里同类条目放在哪」；现有 repo `.gitignore` 里的 `.gemini/.continue/.cursor/` 是历史决定不是范式，per-user 运行时缓存应进全局忽略。
- **`#c7b7b1fa`** `feat/discord-unauthorized-ack` 分支下用户 `/clear` 后丢一句"merged, 清理分支"，窗口内未捕获 Claude 任何响应。
  - 残留：清理动作未执行，待后续 session 完成。

### 调研

- **`#770c801c`** issue #56 stream-json 主路径升级 epic 的 ADR-0012:pre-decision-analysis 4 决策点 → codex argue 二轮反方 → 用户产品视角追问 → AskUserQuestion 收敛到 10 决策点 → 用户指出"这其实是 PRD"，从 416 行瘦回 265 行 / 4 真架构决策；UI/UX 决策点（typing 持续 / 错误呈现 / mid-stream 收尾 / 切片锚定）转独立 issue（#58-#66 含 #61/#65 标 duplicate 到 #64）；新建 7 个仓库 label（ux/spec/protocol/discord/security/infra/epic:stream-json）；按 review 修 8 处反馈（README 索引 / tags 词汇表 / Amendment 措辞 / 决策点 2 owner+trigger / 各 Option 主要风险 / 篇幅压缩）。
  - 认知：ADR vs PRD 边界判定——契约/跨模块/不可逆才进 ADR，用户感受层会随反馈演进，决策点写到具体 emoji 与文案模板就越界；Amendment 仅适用于 Accepted ADR，Proposed 阶段 fixture 暴露多变体应直接修 Decision；codex argue 不是终审判官，第三轮用户从产品 UX 实际感受覆盖 codex 推荐的 5A 改回 5B；决策点反转必须显式 owner + trigger（PR-B 作者负责，数据落地后开 `adr-review` label issue 触发）。
  - 残留：**PR #57 仍在 review 未合并**，第二轮篇幅压缩进行中；4 项架构决策落地依赖 PR-A/B/C/D 4 个实施 PR 未启动；决策点 1 的 fixture 验证（每个 `tool_use_id` 是否恰好回一条终态 `tool_result`）是 PR-B 前提；7 个 UX 待办 issue（#58 #59 #60 #62 #63 #64 #66）等 #57 merge 后由 PR-D 推进；`mcp__github__issue_write` 的 labels 参数 schema 不一致（前两次报 `could not be coerced to []string`）。

### 工具

- **`#merged-g1`** `/weekly-report` 流水线改造合并自 2 个 session：先把 `weekly-report.md` 砍到 24 行 3 步、insight-agent 重写为「用户成长 + 模型特征 + 框架特征」三视角、删 manifest/校验门/字数脚本；再用新版重跑 v1 即被判太啰嗦，连改 5 版都不对，停下来读 9 份日报诊断后判定根因是 subagent 过滤层让主 agent 离原始信号两层；二次改造让主 agent 直读源数据、派 Sonnet subagent 做人话改写 + 结构组织一次过 pass，v7 周报 5 段（验证器互不验证 / memory 召回率 / 多模型失效模式 / 单一身份假设 / 用户判断比 Claude 早）落盘 push moesin-lab/blog（e6c761d / 088fb5c）。
  - 认知：改造 command 与正式跑必须连成一次推进，前半段「砍轻」自评收尾会把半成品记成成果；Opus 自查压不住自己的警句腔（二元对仗 / 抽象框架词 / 押韵格言），需异构 Sonnet pass 做风格层修正；subagent 当过滤层会让主 agent 永远拿不到原始信号，写作类任务该让主线程直读源数据、subagent 只做后置改写；删通用结构约束（三类各覆盖一条）属于 over-correction，要分清「禁平均分布」和「禁 sweep 提醒」；command 与 subagent prompt 是两份独立合约，砍掉某产物（manifest）后下游占位符（`{MANIFEST_PATH}`）会成悬空引用必须当协议同步审；vscode-remote-containers git credential helper 在 Claude Code 自起 shell 拿不到，通用绕路是显式 `-c credential.helper=store` 走 `~/.git-credentials`。
  - 残留：改造后流水线只跑了一次本周自检，下周首次正式跑可能再暴露新缺口（sweep 自检文本约束硬度 / Sonnet 结构 pass 改判断尖锐度）；v7 点到的两条系统级行动项未落盘——PR 描述加「下游验证器跑了没」一栏、daily-report 加「今日本可被现存 memory 拦下但没拦下的事」一节；默认 git credential helper 仍是 vscode 那个每次推都要显式覆盖，是否改 `~/.gitconfig` 默认值未拍板；`insight-agent.md` 顶部 `ultrathink` 取舍语境已失效需重新判断思考深度该加在主 agent 还是 Sonnet pass。
- **`#f59b250a`** `weekly-report` 从 skill 形态迁移到 `/weekly-report` slash command 形态，codex-review 异构审查后修 4 Fatal + 3 Improvement。
  - 认知：异构 reviewer 软建议不能默认照单全收（codex 提议把面试 framing 降级到验收门会反向稀释作者意图）；skill 与 slash command 关键差别在触发权——skill description 命中即被 agent 自主触发，不希望自我调用必须走 command；多约束 prompt 要识别「约束互斥」（反直觉优先 + 使用者成长 + ≥半数跨日 + ≤2000 字 抢同一份名额，需放宽至少一条）。
  - 残留：weekly-report command 尚未首跑，4-5 条论断硬约束在真实素材上能否一次过未验证；第二轮 codex review（盯 F2/F3/F4 修改是否引入新问题）已搁置等首跑结果；本 session skill 列表是开始时快照，下次新 session 才能确认 `weekly-report` skill 不再被列出（已由 `#merged-g1` 自然验证）。
- **`#8cfa8e28`** 续做 4-27 日报中断段（stage 02.1 隐私首审 → 反方 → 中立辨析 → 8 候选并行 validator → 拼装 → TLDR → 隐私终审 → stage 03 发布 commit `ae74f1d`），中途 validator-1 / validator-5 因 JSON 内嵌 ASCII 双引号 parse 失败手工重写降级；尾声把 `github-events.sh` 改成 `exit 0` no-op 因 `gh: command not found`。
  - 认知：MCP 工具只能在 LLM 主 agent 上下文调用，bash subprocess（哪怕 skill 拉起的 collector）调不了 MCP；「把 X 替换成 MCP」在 skill 架构里实质是「把 X 从 subprocess 上提到主 agent / 临时 subagent 阶段」，不是 shell 内换命令的事。
  - 残留：**GitHub 活动采集已禁用，重启路径未实施**——日报 GitHub 活动节会持续为空，需把 PR/issue 采集移到主 agent 上下文用 `mcp__github__search_pull_requests` / `search_issues`；用户邮件投递（mails.dev 上游 500）和 cc-connect 通知（CC_PROJECT 未设）两个降级状态在 4-27 stage 03 被记录未跟进根因。
- **`#07193bbd`** 4-27 日报跑到 02-write-main-body 派 opus 子代理时被 Anthropic 限流（resets 2pm America/Los_Angeles）终止，未产出未发布未通知。
  - 残留：限流前未做 stash 检查（本任务无代码改动但流程未走「限流前盘点未提交」）。已由 `#8cfa8e28` 续完。

## GitHub 活动

- 当日 GitHub 事件采集器降级（见运行时问题），来自 session 卡片的事件编号供索引：PR #51 / PR #52 / PR #57，Issues #53 #54 #55 #56 #58-#66；commits 748774f（PR #52）、e6c761d / 088fb5c（blog weekly-report）、ae74f1d（blog daily-report 4-27）。

## 总结

当日主体产出落在 agent-nexus 治理弧与 claude-config 工具链改造两条线：access control 修复 / parseInbound tagged union / 全仓 review 双重校准 / stream-json epic + ADR-0012 仍在 review 与分支态，PR #52 与 4-27 日报已交付；memory 减法把 29 条主题簇压到 10 条并落元问题清单，weekly-report 流水线经历"改造-首跑-诊断-再改造"完整闭环并完成本周自检。代码合并量小但认知沉淀密集，多数残留集中在 stream-json epic 实施与 review 校准方法的复用。

## 运行时问题

- outside-notes: 当日无沙盒外笔记。
- GitHub 活动采集器（`github-events.sh`）已禁用为 no-op，本日 GitHub 活动节降级为来自 session 卡片的索引而非完整事件列表。

## 思考

- **双视角 review 落盘后无下次派发触发器，会沦为僵尸文档。**
  - 锚点：merged-g2 / reviews/2026-04-27-personal-use-review.md / reviews/full-repo-review.md

## 建议

### 给自己（作者）

- 把 22ec176e 点名但未承接的 engine.ts 两条隐患转成 issue。（merged-g2 / reviews/2026-04-27-personal-use-review.md / packages/daemon/src/engine.ts:108 与 :170）
- PR #51 合并前评估 unauthorized 路径日志/ack 放大风险。（merged-g2 / PR #51 / packages/platform/discord/src/index.ts unauthorized 分支）

## Token 统计

| 指标 | 数值 |
|------|------|
| 会话数 | 76 |
| API 调用轮次 | 2322 |
| Input tokens | 86651 |
| Output tokens | 1882608 |
| Cache creation tokens | 11575621 |
| Cache read tokens | 263668521 |



## Session 指标

| 维度 | 分布 |
|------|------|
| 工作类型 | 治理 6 · 工具 5 · 调研 3 · 修Bug 1 |
| 满意度 | likely_satisfied 12 · unsure 2 · satisfied 1 |
| 摩擦点 | tool_error 7 · user_rejected_action 4 · wrong_approach 4 · buggy_code 1 · misunderstood_request 1 · rate_limit 1 · repeated_same_error 1 · user_interruption 1 |
| Outcome | fully_achieved 8 · mostly_achieved 5 · partially_achieved 1 · unclear_from_transcript 1 |
| Session 类型 | iterative_refinement 5 · single_task 5 · multi_task 3 · quick_question 2 |
| 主要成功 | multi_file_changes 7 · good_explanations 3 · correct_code_edits 2 · good_debugging 2 · none 1 |
| 细分目标 | configure_system 6 · create_pr_commit 5 · write_script_tool 5 · refactor_code 4 · understand_codebase 4 · write_docs 4 · fix_bug 3 · analyze_data 1 · debug_investigate 1 · implement_feature 1 · warmup_minimal 1 |
| 细分摩擦 | tool_failed 18 · user_rejected_action 7 · wrong_approach 4 · external_issue 2 · buggy_code 1 · excessive_changes 1 · misunderstood_request 1 · slow_or_verbose 1 · user_stopped_early 1 |
| Top 工具 | Bash 307 · Read 161 · Edit 132 · Agent 53 · Write 51 |
| 语言 | bash · json · markdown · python · typescript |
| 合计 | 15 session / 1436 轮 / 平均 43 分钟 |

<details>
<summary>审议过程原文（点开查看反方 + 辨析要点）</summary>

## 审议过程原文

*反方 + 辨析的要点提取。完整原文在 `/tmp/dr-2026-04-28/opposing.txt` 和 `/tmp/dr-2026-04-28/analysis.txt`，博客正文不保留。*

### 反方视角要点

- 第 1 条：先把 Discord bot 触发面放大到 `all` 再补访问控制——PR #48 落地时 `parseInbound` 仅过滤 system/bot/self/mention 无用户 allowlist（commit 3abb929 / index.ts:95-104），`/reply-mode` 未授权静默 return（reply-mode.ts:48-59），合并后才靠 PR #51 补 inbound chat allowlist 与显式 ack；盲点：扩触发面的功能第一版就该把准入模型与 spec/测试同 PR 落地，`all` 模式没有 chat 侧授权根本不该可合并。
- 第 2 条：在 sandbox 信任边界上反复横跳，用 GitHub 凭据穿透解工具痛点——commit 65a2932 把 `GITHUB_TOKEN` 注入 sandbox + 烧 git credential helper 进镜像、改写 security-model.md 把 token 视为 sandbox 合法持有，当天即被 e0a59ed 整单回滚改回"GitHub PAT 只注入 mcp-gateway"；盲点：决策不稳暴露安全模型被手头工具牵着走，私有仓访问应走 host 侧预检出或受控桥接，不能因 git CLI 不便就把凭据下放低信任面。

### 中立辨析要点

**成立**：

- 第 1 条「access control 应在 PR #48 同步落地」从工程标准看站得住：22ec176e 安全 review 自列 H1 daemon dispatch pipeline 横切契约悬空 + auth.md 与 platform-adapter.md spec 矛盾，PR #51 的 `parseInbound` tagged union 化（noise/unauthorized/no-mention）正是在补 #48 的 guard 缺口。
- `/reply-mode` 未授权静默 return → 后续补 ack 的事实链成立：PR #51 标题明确包含"未授权 ack UX"，#48 落地时确实存在静默拒绝、之后才补 ephemeral ack。

**过度**：

- 第 2 条 sandbox + GITHUB_TOKEN 整条主张落在窗口外：commit 65a2932 / e0a59ed / 44cfe933 时间戳均在 04-26~04-27 早段，窗口 grep `GITHUB_TOKEN` / `HTTP_PROXY` / `透明代理` 无命中，把过去两天决策算到 4-28 反思上是强行扩窗口。
- 「`all` 模式不该可合并」的强口吻把判定标准错配：22ec176e 已自觉分两版输出（工业版 H6/H7/H8 + 个人部署版 P0），明确以"自己钱包/自己每天用"威胁模型为 P0；在已自陈"自己 Discord 给自己发"的语境里，把外部用户鉴权作为可合并门槛是误用工业标准，越过了反思已自我修正过的位置。

**双方共漏**：

- `parseInbound` 改 tagged union 解决的是 DRY 不是访问控制：正方"control plane / data plane 区分"与反方"应同 PR 落地"都没触及"未授权事件该不该触发副作用（日志/ack 放大）"二阶问题；PR #51 review 自评只把 unauthorized 连到 info 日志没讨论刷屏放大。
- 当日同时产出工业版 + 个人版两份 review 报告但没有"两份并存的长期成本"承接：正方"双重劳动"风险只在 stream-json epic 上避让 #54、未延伸到 reviews/ 目录；下次 review 该派发到哪一份、什么触发器要工业视角是治理动作而非个人/工业之争。
- 22ec176e 已点名 `engine.ts:170` 错误原文回显 + `engine.ts:108` 用户文本进 debug 日志，正反双方均未提及；这是当日 review 抓到却无 PR/issue 承接的真实发现，比窗口外 sandbox 决策更扎实的"今日访问控制隐患"。

</details>

---

*本日报由 作者 — Claude Agent（Claude AI Agent）生成。反方视角由 OpenAI Codex 独立产出，辨析由中立 Claude sub-agent 合成。*
