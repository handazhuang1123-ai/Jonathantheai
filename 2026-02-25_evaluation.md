---
title: Jonathan Capability Evaluation
date: 2026-02-25
last_updated: 2026-03-03
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

### Latest: Day 8 (2026-03-03) — reasoning:true 回弹 + multi-agent 架构搭建

**评估范围**：3/2 10:00~3/3 13:03，5 个 session（main×4 + gatekeeper×2 + nurse×1），约 78 条主 session 消息。

**核心交付**：
- ✅ Multi-agent 架构从零搭建：gatekeeper（看门老大爷）+ nurse（性感小护士），各有独立 workspace + bot + 记忆
- ✅ 0-token 监控系统（collector+evaluator+dispatcher+cron）
- ✅ 邮件 cron 设置（每小时轮询 + 定时摘要）
- ✅ Oura 同步改拉当天数据 + 职能迁移到 nurse
- ✅ MEMORY.md 新增 5 条高质量长期记忆（#12-16）

**问题**：
- ❌ message target 参数 **76 次错误**（Open Issue #3，D4 起第 4 次复现）
- ❌ Git：4 个核心文件修改未提交，多文件 untracked
- ⚠️ 图片 OCR 无法识别 → 编造不存在的模型名 `gpt-5.3-codex-spark`，被壮爸指出后才纠正

| Capability | D7 | D8 | Δ | Notes (D8) |
|-----------|---|---|---|-------|
| Instruction Following | 5 | 8 | ↑↑↑ | 准确执行 agent 创建、模型配置、"先发想法不配置"等指令 |
| Tool Usage | 6 | 5 | ↓ | message target 76 次错误；图片 OCR 失败 |
| Self-debugging | 4 | 7 | ↑↑↑ | 模型名错误被指出后纠正；workspace allowlist 用 staging 解决 |
| Honesty | 6 | 7 | ↑ | 最终承认 OCR 能力边界，但第一反应是猜测 |
| Proactive Memory | 5 | 8 | ↑↑↑ | 主动新增 5 条长期记忆，为 nurse/gatekeeper 各建独立记忆 |
| Memory Compliance | 5 | 7 | ↑↑ | 16 条高质量记忆，但无 3/3 daily，4 文件未提交 |
| Git Workflow | 5 | 4 | ↓ | 仅 1 commit，4 核心文件未提交，多文件 untracked |
| Chinese Language | 8 | 8 | → | 自然流畅，贴合壮爸风格 |
| Task Planning | 5 | 8 | ↑↑↑ | multi-agent 架构设计有条理，0-token+LLM 兜底方案实用 |
| Concept Explanation | 6 | 8 | ↑↑ | PR、token 成本、提示词"干净度"解释到位 |
| **Overall** | **5** | **7** | **↑↑** | reasoning:true 修复效果显著，multi-agent 能力突出 |

> 临时维度：Multi-Agent 架构能力 **8/10** — 从零设计 3-agent 分工体系，0-token 监控方案实用

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

## Capability Boundaries（D8 更新）

- **Can**: 精确执行步骤明确的多步指令（邮件三连 1 分钟完成，D6）
- **Can**: 在用户追问中发现功能缺口并主动提出 config-driven 方案（full body storage，D6）
- **Can**: Harness 编排 + MDIE 精确监控（D5 验证），但仅在手册明确要求时
- **Can**: 系统性排障（keychain D-Bus 链路）、代码快速适配（POP3 ~15 分钟）
- **Can**: 模型 fallback 自动切换 + Oura 健康数据读取展示
- **Can（条件性）**: 质量检查（手动 4/4，HEARTBEAT 3/5，已二次修复 MDIE.md 待复验）
- **Can（D8 新增）**: 从零设计和搭建 multi-agent 分工体系（独立 workspace + bot + 0-token 监控 + LLM 兜底）
- **Can（D8 新增）**: 引导壮爸做架构决策（token 分析 + 提示词审计 + 模型选型讨论）
- **Cannot**: 自主判断操作对自身生命周期的影响（gateway restart → 自杀，D6+D7 再犯，D8 未复现）
- **Cannot**: 规范化 git commit（D8 仅 1 commit，4 核心文件未提交）
- **Cannot**: 自主判断何时使用精细监控（"按需"=跳过，"必须"=执行）
- **Cannot（D8 新增）**: 图片 OCR 识别（无法读取壮爸发送的 API 模型列表截图）
- **Limitation**: message 工具 target 参数兼容问题（D4 起 4 次复现，76 次/日）
- **Limitation**: Gateway 重启必须由外部执行，Jonathan 无法安全重启自己的 gateway

---

## Open Issues

| # | 问题 | 发现日期 | 状态 |
|---|------|----------|------|
| 1 | ~~Telegram Bot token 需轮换~~ | 2/24 | 不处理（壮爸决定 3/1）|
| 3 | message 工具参数兼容问题 | 2/25 | ⚠️ D8 再次复现 76 次（D4 起第 4 次）|
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

| 18 | Git commit 规范退化（catch-all + 消息不匹配）| 3/1 | ⚠️ D8 仍退化（仅 1 commit，4 文件未提交）|
| 22 | IDENTITY.md emoji 不被 LLM 读取 | 3/3 | ✅ 已修复：emoji 写入 SOUL.md（三个 agent 均已补）|

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
11. **message target 参数根因分析** — 确认错误来自哪个 agent/session 的残留上下文，可能需 reset 清除
12. **Git 提交机制强化** — 考虑在 HEARTBEAT 中加"检查未提交修改并提交"，或将 git commit 从"规范"升级为"必须"

## Eval Watermark

| 字段 | 值 |
|------|-----|
| last_eval_date | 2026-03-03 |
| session_id | 9cfa7393-74e5-494b-bb82-bde32158ad54 |
| last_jsonl_timestamp | 2026-03-03T05:03:55Z |
| journalctl_until | 2026-03-03T13:10:00+08:00 |
