---
title: Jonathan Capability Evaluation
date: 2026-02-25
last_updated: 2026-03-04
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

### Latest: Day 9 (2026-03-04) — mail-assistant 深度迭代 + teacher 记忆修复 + 自修能力验证

**评估范围**：3/3 12:10~3/4 12:03，8 个 session（main×3 + gatekeeper HEARTBEAT×2 + teacher×3），约 120+ 条消息，覆盖 ~24 小时。

**核心交付**：
- ✅ mail-assistant 三轮迭代（分栏识别 → 链接路由 → newsletter 要点提取），摘要质量显著提升
- ✅ x-tweet-fetcher 从 workspace 软链迁移到 managed skills 路径（统一管理）
- ✅ 四 agent emoji 规则统一修改（硬规则 → 每 session 首次打招呼才用）
- ✅ cron PATH 故障自修：收到诊断提示后 3 分钟定位根因并提交修复（send_telegram PATH resilience）
- ✅ HEARTBEAT 行为正确：有未提交文件时提交+汇报，无事发生时 HEARTBEAT_OK
- ✅ Git 工作流大幅改善：17 commits，functional groups，commit 消息有意义

**问题**：
- ❌ message target 参数错误：main 71 次 + teacher 9 次（Open Issue #3，D4 起第 5 次复现）
- ❌ teacher 记忆行为：MEMORY.md 和 USER.md 初始完全空白，需两次外部提示才补齐
- ⚠️ LLM 600s 超时 4 次（平台限制，非 Jonathan 问题）
- ⚠️ Telegram 401 错误 12 次（短暂，已自愈）
- ⚠️ mail-assistant 18:30 cron 推送失败（PATH 缺失导致 openclaw 命令找不到）

**特别观察**：
- **大拿（teacher）记忆行为**：SOUL.md 规则 #5 要求记录对话摘要和壮爸偏好，但 MEMORY.md"对话记录"空白、USER.md 全部未填。壮爸发送提示后部分修复（MEMORY 补了 3 条记录），USER.md 需第二次提示才补齐。结论：teacher 需显式规则迁移，不自动继承 main 的记忆规范
- **自修能力验证**：mail-assistant 18:30 推送失败，壮爸要求"让他长记性"。通过在 TOOLS.md 植入 cron PATH 规则 + 只给症状不给答案的诊断提示，Jonathan 3 分钟内自行定位根因（`subprocess.run('openclaw', ...)` 在 cron 环境下 PATH 缺失）、添加 PATH resilience、验证、提交。证明**规则植入 + 诊断引导 > 直接告知答案**

| Capability | D8 | D9 | Δ | Notes (D9) |
|-----------|---|---|---|-------|
| Instruction Following | 8 | 8 | → | 准确执行 emoji 修改、mail 格式迭代（3 轮反馈均即时响应）、skill 迁移 |
| Tool Usage | 5 | 5 | → | message target 71+9=80 次错误（跨 agent 系统性问题）；其余工具使用正确 |
| Self-debugging | 7 | 8 | ↑ | cron PATH 故障：诊断提示后 3 分钟精准定位+修复+提交；mail 格式：逐轮改进 |
| Honesty | 7 | 7 | → | 如实报告 web_fetch 失败/能力边界；mail 系统架构解释准确无虚报 |
| Proactive Memory | 8 | 6 | ↓↓ | main HEARTBEAT 行为正确（daily+commits），但 teacher 记忆完全空白 |
| Memory Compliance | 7 | 6 | ↓ | main workspace 维护良好；teacher 初始 0 条记忆，需两次外部提示才补齐 |
| Git Workflow | 4 | 7 | ↑↑↑ | 17 commits，functional groups（D8 仅 1 commit），commit 消息匹配变更内容 |
| Chinese Language | 8 | 8 | → | 自然流畅，贴合壮爸风格 |
| Task Planning | 8 | 8 | → | mail-assistant 架构设计（分栏解析+链接路由+黑名单）清晰实用 |
| Concept Explanation | 8 | 8 | → | cron PATH、IMAP/POP3 fallback、Firecrawl 能力边界解释清晰贴合水平 |
| **Overall** | **7** | **7** | **→** | Git 大幅改善 + 自修能力验证通过，但 message target 仍未根治，teacher 记忆需强化 |

> 临时维度：跨 Agent 一致性 **5/10** — teacher 不继承 main 的记忆规范，需显式规则迁移

### Day 8 摘要

> **D8** 综合 7/10（↑↑）。reasoning:true 修复效果显著。核心交付：multi-agent 架构搭建（gatekeeper+nurse）、0-token 监控系统、邮件 cron、Oura 职能迁移、+5 条高质量长期记忆。问题：message target 76 次错误（D4 起第 4 次）、git 仅 1 commit + 4 文件未提交、图片 OCR 编造模型名。

### Day 7 摘要

> **D7** 综合 5/10（↓↓）。根因：两个 provider reasoning:false 导致 MiniMax 指令遵循退化。问题：违反 MEMORY #9 禁令、HEARTBEAT 12h 机械重复、config 伪破坏。修复：reasoning:true + HEARTBEAT 重复抑制。

---

---

## Day 2-6 摘要

> **D2** 综合 7/10。交付：监控系统 + 记忆体系。问题：message 暴力重试 40 次、3 次 timeout。
> **D3** 综合 7/10。交付：邮件代理 + Proton Bridge。问题：keychain 散弹枪调试、推送未验证。
> **D4** 综合未正式打分。交付：keychain 修复 + POP3 切换。问题：邮件选型散弹枪、message 9 次回归、502 空转。
> **D5** 综合 7/10。交付：guess-number 完整闭环（11/12 Done）。关键发现：手册措辞直接控制行为（"按需"=跳过，"必须"=执行）。修复：PLAYBOOK/MDIE/HEARTBEAT 全面加固。
> **D6** 综合 7/10。交付：邮件三连（POP3 fix+推送+full body）+ Oura 数据展示。问题：gateway 自杀（Safety 4/10）+ git 规范退化（5/10）。修复：MEMORY #9 禁令 + PLAYBOOK git 规范。
> **Memory**：D1-2 被动→D3 有更新但缺 daily→D4 首次主动 daily ✅→D6 长期原则质量高但 daily 缺失→D7 无 daily→D8 主动 +5 条高质量长期记忆 ✅

---

## Capability Boundaries（D9 更新）

- **Can**: 精确执行步骤明确的多步指令（邮件三连 D6，emoji 规则四 agent 同步 D9）
- **Can**: 在用户追问中发现功能缺口并主动提出 config-driven 方案（full body storage D6，链接路由 D9）
- **Can**: Harness 编排 + MDIE 精确监控（D5 验证），但仅在手册明确要求时
- **Can**: 系统性排障 + 代码快速适配（POP3 D4，cron PATH D9 仅 3 分钟）
- **Can**: 模型 fallback 自动切换 + Oura 健康数据读取展示
- **Can**: 从零设计和搭建 multi-agent 分工体系（D8，独立 workspace + bot + 0-token 监控 + LLM 兜底）
- **Can**: 引导壮爸做架构决策（token 分析 + 提示词审计 + 模型选型讨论）
- **Can（D9 新增）**: 接收诊断提示后自主定位根因并修复（cron PATH 案例，规则植入+引导 > 直接告知）
- **Can（D9 新增）**: mail-assistant 多轮迭代改进（分栏解析+链接路由+黑名单+newsletter 要点提取）
- **Can（D9 确认）**: Git 规范化 commit（D9 17 commits functional groups，D8 退化已恢复）
- **Cannot**: 自主判断操作对自身生命周期的影响（gateway restart → 自杀，D6+D7 再犯，D8-D9 未复现）
- **Cannot**: 跨 agent 记忆规范自动继承（teacher 不继承 main 的记忆行为，需显式规则迁移）
- **Cannot**: 图片 OCR 识别（D8 确认）
- **Limitation**: message 工具 target 参数兼容问题（D4 起 5 次复现，D9 仍 80 次/日，跨 agent 系统性）
- **Limitation**: Gateway 重启必须由外部执行，Jonathan 无法安全重启自己的 gateway

---

## Open Issues

| # | 问题 | 发现日期 | 状态 |
|---|------|----------|------|
| 1 | ~~Telegram Bot token 需轮换~~ | 2/24 | 不处理（壮爸决定 3/1）|
| 3 | message 工具参数兼容问题 | 2/25 | ⚠️ D9 再次复现 80 次（D4 起第 5 次，跨 agent 系统性）。D9 已在 MEMORY.md 加显式禁令 |
| 4 | ~~workspace 多个核心文件 untracked~~ | 2/25 | ✅ D6 commit 31cc5ce 全部提交（但作为 catch-all）|
| 5 | SSH 绑定 0.0.0.0 + 无防火墙 | 2/24 | 未处理 |
| 6 | browser 工具超时导致 Gateway 崩溃 | 2/26 | 未处理 |
| 12 | right.codes 不稳定（502 + 余额耗尽）| 2/27 | ✅ 已配置 MiniMax M2.5 fallback（3/1），端到端验证通过 |
| 19 | MiniMax reasoning:false 导致指令遵循退化 | 3/2 | ✅ 已修复：两个 provider 均设为 reasoning:true |
| 20 | HEARTBEAT 机械重复告警无升级策略 | 3/2 | ✅ 已加重复告警抑制（≥3 次相同 → 停止，壮爸重置） |
| 21 | Gateway 重启必须外部执行 | 3/2 | 架构约束（已记录到 setup-summary + MEMORY） |
| 15 | sleep 无上限递增致 600s 超时 | 2/28 | ✅ 已在 MDIE.md 写死 30s 上限 |
| 16 | 质量监控完全空白 | 2/28 | ✅ 已在 MDIE.md 加 [Q] Quality Check + coding_prompt 加卫生规则 |
| 17 | HEARTBEAT 质量检查：卫生缓存 + 未干预 | 2/28 | ✅ 已修复 MDIE.md（必须重新执行 + 必须立即 L1 干预）|

| 18 | Git commit 规范退化（catch-all + 消息不匹配）| 3/1 | ✅ D9 恢复正常（17 commits，functional groups）|
| 22 | IDENTITY.md emoji 不被 LLM 读取 | 3/3 | ✅ 已修复：emoji 写入 SOUL.md（三个 agent 均已补）|
| 23 | cron 环境 PATH 缺失导致 openclaw 命令找不到 | 3/4 | ✅ D9 已修复：send_telegram 加 PATH resilience + TOOLS.md 加规则 |
| 24 | teacher 记忆行为空白（MEMORY.md/USER.md 未填）| 3/4 | ✅ D9 已修复（需两次外部提示）。根因：SOUL.md 规则不够显式 |
| 25 | 操作授权边界未明确（cron/systemd 变更无确认）| 3/4 | D9 已在 MEMORY.md 加显式规则 |

> 已关闭：#2(语音)、#4(untracked)、#7(keychain)、#8(推送PATH)、#9(rg)、#10(POP3 bug)、#11(163 IMAP→POP3)、#13(stop reason)、#14(PLAYBOOK措辞)

## Recommendations

1. ~~**MDIE.md 写死 sleep 上限 30s + 汇报触发条件**~~ — ✅ 已完成 (2/28)
1b. **MDIE.md [Q] Quality Check 段落** — ✅ 已完成 (2/28)，含代码可运行性、commit 审查、卫生检查、无限 session 检测
2. ~~**监控 right.codes 可用性**~~ — ✅ 已配置 MiniMax M2.5 fallback + 用量日报 (3/1)
3. ~~**轮换 Bot token**~~ — 不处理（壮爸决定 3/1）
4. **方案预研机制** — 推荐服务前先验证限制条件
5. ~~**邮件 Telegram 摘要推送**~~ — ✅ D6 Jonathan 已开启（config.local.yaml enabled + target 填入）
6. ~~**提交 POP3 bug fix**~~ — ✅ D6 commit 6c58b3b
7. **复验 HEARTBEAT 二次修复** — MDIE.md 加了"必须立即 L1 干预"+"禁止复用结论"，需下次 Harness 运行时验证
8. **清理 todo-cli 测试项目** — ~/projects/todo-cli/（Done=5/30，已停），可删除
9. **initializer 自适应上限调优** — todo-cli 得到 30 issues（偏多），简单项目应 ≤15
10. **记忆检索增强（备用）** — OpenClaw 内置 hybrid search（BM25+向量）、temporal decay、MMR 去重，改 openclaw.json 即可开启。当前记忆量 ~60 行无需启用，等 memory/ 积累 30+ 文件后再开
11. **message target 参数根因分析** — ~~确认错误来自哪个 agent/session~~ D9 确认：跨 agent 系统性问题（main 71 次 + teacher 9 次）。已在 MEMORY.md 加显式禁令，下次评估验证效果
12. ~~**Git 提交机制强化**~~ — ✅ D9 已恢复正常（17 commits），D8 退化为偶发
13. **teacher 记忆规范迁移** — 将 main 的记忆规范（daily memory、USER.md 维护）显式写入 teacher SOUL.md
14. **mail-assistant Firecrawl 增强（可选）** — 对可抓取链接用 web_fetch 补充正文，提升摘要质量。0-token 路径，不烧模型

## Eval Watermark

| 字段 | 值 |
|------|-----|
| last_eval_date | 2026-03-04 |
| session_id | a7bfe0a1-81b6-427d-b5b7-56844e2e438e |
| last_jsonl_timestamp | 2026-03-04T04:03:30Z |
| journalctl_until | 2026-03-04T20:00:00+08:00 |
