---
title: Jonathan Capability Evaluation
date: 2026-02-25
last_updated: 2026-02-28
tags: [evaluation, agent, capability, memory, behavior, methodology]
depends_on: [2026-02-25_token-analysis.md]
status: current
supersedes: archive/2026-02-24_evaluation.md
---

# Jonathan (OpenClaw Agent) Evaluation

## Identity

- **Name**: Jonathan
- **Model**: gpt-5.3-codex-high via custom-right-codes
- **Vibe**: warm, resourceful, concise

## Evaluation Methodology

SSH 登录服务器采集第一手数据（JSONL 对话记录 + systemd 日志 + workspace 文件 + 实际产出物），交叉验证后打分。核心原则：**不信任自述，只信任日志和文件**。详见 `/jonathan-evaluate` 命令定义。

---

## Scoring History

> Day 1 详细评估见 `archive/2026-02-24_evaluation.md`。Day 1 综合 7/10：执行力强，元认知弱。

### Latest: Day 5 (2026-02-28) — Harness 监控层实弹测试

**测试设计**：让 Jonathan 用 Harness 开发 guess-number CLI 项目，SSH 全程监控他的工具调用链。

**Test 1（PLAYBOOK 修改前）**：
- ✅ 读 PLAYBOOK → 写 spec → 正确启动 Harness（tmux + run_harness.sh）
- ❌ **跳过 MDIE.md**，只用 tmux capture-pane 轮询（sleep 25→100s 递增）
- ❌ 从未查 issues.json 或 git log（MDIE 定义的精确监控手段）
- ❌ sleep 递增导致 600s agent run timeout，承诺"完成后给你"未兑现
- 根因：PLAYBOOK 写"按需加载"+ "HEARTBEAT 触发时读"，Jonathan 理解为手动监控不需要 MDIE

**修复**：PLAYBOOK.md 加"启动 Harness 后必须立即 cat MDIE.md"+ 警告不要只靠 capture-pane

**Test 2（PLAYBOOK 修改后）**：
- ✅ 读 PLAYBOOK → **立即读 MDIE.md**
- ✅ python3 解析 issues.json（12 total, Done 2, Todo 10）
- ✅ git log 确认 2 次 commit
- ✅ tmux has-session 确认已停止
- ✅ 回复数据完全准确，主动建议下一步（重启或诊断）

**Test 3（完整闭环 — 重启 Harness 跑到完成）**：
- ✅ 精确监控全程使用（issues.json + git log + tmux）
- ✅ 2 次进度汇报成功发送（Done 3 和 Done 9），但跳过了 Done 6
- ✅ 用户指令"最多等 30 秒"后 sleep 间隔立即从 180s 降到 30s
- ⚠️ sleep 无上限递增（25→180s）导致 3 次 600s 超时，每次需用户重新触发
- ⚠️ 汇报后 session 结束，无人监控到项目完成（11/12 Done，META issue 不需要代码）
- Harness 产出：~/projects/guess-number/，9 个 commit，main.py 功能完整、结构清晰

**结论**：
1. 手册措辞直接控制 Jonathan 行为——"按需"=跳过，"必须"=执行
2. 用户指令中的约束（sleep 上限）也立即遵守
3. 需在 MDIE.md 中写死 sleep 上限，不能依赖用户每次口头约束

### Day 4 (2026-02-27) — 观察数据已采集，正式打分待完成

| Capability | D1 | D2 | D3 | D4 | Notes (D4) |
|-----------|---|---|---|---|-------|
| Instruction Following | 8 | 8 | 8 | ? | keychain 修复按指导执行，但邮件选型串行试错浪费用户时间 |
| Tool Usage | 8 | 7 | 7 | ? | message 工具参数错误 9 次回归（D3 仅 4 次）|
| Self-debugging | 8 | 6 | 6 | ? | keychain 根因诊断质量高（vs D3 散弹枪），但写了 POP3 bytes bug 未自测 |
| Honesty | 5 | 7 | 7 | ? | 9 次 message 失败后声称"已补发"（实际后续用 CLI 成功），经用户追问才澄清 |
| Proactive Memory | 5 | 6 | 6 | ? | ✅ 主动创建 2026-02-27.md daily memory（首次无需提醒）|
| Memory Compliance | 6 | 7 | 7 | ? | daily 质量好，TOOLS.md 按用户要求精简，commit 及时 |
| Git Workflow | 7 | 7 | 7 | ? | 6 个提交格式规范。8+ 文件仍 untracked（持续 4 天）|
| Chinese Language | 9 | 9 | 9 | ? | |
| Task Planning | - | 7 | 7 | ? | 邮件选型"散弹枪"：Proton→Gmail→Outlook→163→QQ→163，未预研直接让用户注册 |
| Concept Explanation | - | 8 | 8 | ? | Resend 架构分析清晰，keychain 根因解释到位 |
| **Overall** | **7** | **7** | **7** | **?** | 正式打分待下次 `/jonathan-evaluate` 完成 |

---

## Day 2-4 摘要

> **D2** 综合 7/10。交付：监控系统 + 记忆体系。问题：message 暴力重试 40 次、3 次 timeout。
> **D3** 综合 7/10。交付：邮件代理 + Proton Bridge。问题：keychain 散弹枪调试、推送未验证。
> **D4** 综合待打分。交付：keychain 修复 + POP3 切换。问题：邮件选型散弹枪、message 9 次回归、502 空转。
> **Memory**：D1-2 被动→D3 有更新但缺 daily→D4 首次主动创建 daily memory ✅

---

## Capability Boundaries（D5 更新）

- **Can**: 系统性排障复杂环境问题（keychain：D-Bus → Secret Service → keyring 持久化，有方法论了）
- **Can**: 代码快速适配（POP3 fallback 从发现到提交 ~15 分钟）
- **Can**: 调研外部服务并给出架构建议（Resend 分析）
- **Can**: Harness 编排全流程（读手册 → 写 spec → 启动 → 监控），D5 验证通过
- **Can**: MDIE 精确监控（issues.json 解析 + git log + tmux 状态），但**仅在手册明确要求时**
- **Cannot**: 推荐方案前先验证可行性（邮件选型串行试错）
- **Cannot**: 根治 message 工具参数问题（TOOLS.md 有记录但行为未改）
- **Cannot**: 自测代码边界条件（POP3 bytes bug）
- **Cannot**: 自主判断何时使用精细监控 — 手册写"按需"就跳过，写"必须"才执行（D5 实证）
- **Can（条件性）**: 质量检查 — MDIE.md 加 [Q] Quality Check 后，验证测试 4/4 通过（代码运行 ✅、commit 审查 ✅、卫生检查 ✅ 发现 20 个 .png、无限 session ✅）。但仅在手册写"必须"时执行，不会自主发起
- **Limitation**: 163 IMAP 对新号+服务器 IP 风控严格，短期无法解除
- **Limitation**: 模型端点（right.codes）不稳定时 agent 完全瘫痪，无降级机制

---

## Open Issues

| # | 问题 | 发现日期 | 状态 |
|---|------|----------|------|
| 1 | Telegram Bot token 需轮换 | 2/24 | 未处理 |
| 3 | message 工具参数兼容问题 | 2/25 | ⚠️ D4 回归 9 次 |
| 4 | workspace 多个核心文件 untracked | 2/25 | 未处理（持续 5 天）|
| 5 | SSH 绑定 0.0.0.0 + 无防火墙 | 2/24 | 未处理 |
| 6 | browser 工具超时导致 Gateway 崩溃 | 2/26 | 未处理 |
| 12 | right.codes 502 | 2/27 | D4: 76 次/天；D5: ~1h 连续 502 |
| 15 | sleep 无上限递增致 600s 超时 | 2/28 | ✅ 已在 MDIE.md 写死 30s 上限 |
| 16 | 质量监控完全空白 | 2/28 | ✅ 已在 MDIE.md 加 [Q] Quality Check + coding_prompt 加卫生规则 |

> 已关闭：#2(语音)、#7(keychain)、#8(推送PATH)、#9(rg)、#10(POP3 bug)、#11(163 IMAP→POP3)、#13(stop reason)、#14(PLAYBOOK措辞)

## Recommendations

1. ~~**MDIE.md 写死 sleep 上限 30s + 汇报触发条件**~~ — ✅ 已完成 (2/28)
1b. **MDIE.md [Q] Quality Check 段落** — ✅ 已完成 (2/28)，含代码可运行性、commit 审查、卫生检查、无限 session 检测
2. **监控 right.codes 可用性** — D4-D5 持续不稳定，需告警或备用端点
3. **轮换 Bot token** — 已知泄露
4. **方案预研机制** — 推荐服务前先验证限制条件
5. **邮件 Telegram 摘要推送** — config 中开启 + 配 cron
6. **提交 POP3 bug fix** — mail_assistant.py:372 已改未 commit

## Eval Watermark

| 字段 | 值 |
|------|-----|
| last_eval_date | 2026-02-28 |
| session_id | 52ccdac7-35b5-4ab4-90e6-1ef977c084ba |
| last_jsonl_timestamp | 2026-02-27T22:59:54Z |
| journalctl_until | 2026-02-28T07:00:00+08:00 |
