---
title: Jonathan Capability Evaluation
date: 2026-02-25
last_updated: 2026-02-26
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

### 数据来源

评估基于以下第一手数据（非 Jonathan 自述）：

1. **SSH 登录服务器**（`ssh zhuangba@192.168.0.18`），直接读取：
   - 会话 JSONL 文件（`~/.openclaw/agents/main/sessions/*.jsonl`）— 完整对话记录
   - systemd journal 日志（`journalctl --user -u openclaw-gateway`）— 工具调用、错误、超时
   - workspace 文件变更（`~/.openclaw/workspace/` git log + 文件内容）
   - 实际产出物（脚本、报告、git 提交）
2. **sessions.json** — token 消耗、cache 统计、model 配置

### 评估流程

```
SSH 进入服务器 → 提取对话记录 → 提取错误日志 → 检查实际产出 → 交叉验证 → 打分
```

> 核心原则：**不信任 Jonathan 的自述，只信任日志和实际文件**。

---

## Scoring History

> Day 1 详细评估见 `archive/2026-02-24_evaluation.md`。Day 1 综合 7/10：执行力强，元认知弱。

### Latest: Day 2 (2026-02-25)

| Capability | D1 | D2 | Δ | Notes |
|-----------|---|---|---|-------|
| Instruction Following | 8 | 8 | → | 完成监控系统，但 healthcheck 被中断后未恢复 |
| Tool Usage | 8 | **7** | ↓ | message 参数错误 **40 次连续失败** |
| Self-debugging | 8 | **6** | ↓ | 40 次重试相同错误，未换策略 |
| Honesty | 5 | **7** | ↑ | 承认中途停了、承认没写 memory |
| Proactive Memory | 5 | **6** | ↑ | 仍需提醒，但提炼质量高 |
| Memory Compliance | 6 | **7** | ↑ | MEMORY.md（4 条）+ daily 文件 |
| Git Workflow | 7 | 7 | → | 成功创建 GitHub 仓库并推送 |
| Chinese Language | 9 | 9 | → | |
| Task Planning | - | **7** | 新 | 拆解合理，卡住时沟通节奏差 |
| Concept Explanation | - | **8** | 新 | skill/agent/memory/context 讲解准确 |
| **Overall** | **7** | **7** | → | 能力面更广，工具鲁棒性是新短板 |

---

## Day 2 详细观察

### 交付成果

1. **服务器健康监控系统**（`~/monitor/`）
   - `health-check.sh` — CPU/内存/磁盘/mihomo/openclaw/noVNC/订阅，输出 JSON + Markdown
   - `alert.sh` — 异常时通过 OpenClaw 通道发 Telegram 告警
   - cron 每天 08:00 执行
   - GitHub 仓库：`Jonathantheai/monitor-2026-02-25`（public, MIT license）
2. **长期记忆体系建立**
   - MEMORY.md 写入 4 条原则（身份解耦、能力≠授权、记忆存决策、附件用绝对路径）
   - memory/2026-02-25.md 日志

### 正面行为

- 监控脚本质量不错：结构化 JSON + 人类可读摘要双输出
- 原则提炼能力出色：从具体项目经验抽象到"身份与任务解耦"等高阶原则
- 对 OpenClaw 内部机制理解深入：准确解释 context window、compaction、pruning
- 诚实度改善：被问"任务停了还是没做完"时直接承认

### 严重问题

1. **message 工具反复失败（40 次）**
   - 错误信息 `Use 'target' instead of 'to'/'channelId'` 反复出现
   - 集中在 11:27-11:37（约 30 次）和 11:42-11:44（约 10 次）
   - **核心问题**：未读取错误信息调整参数，用相同错误参数暴力重试
   - 违反 SOUL.md "Be resourceful before asking" 原则

2. **3 次 embedded run timeout（600s/10 分钟）**
   - 08:06 — 健康检查执行卡住
   - 08:58 — 仓库创建探测超时
   - 11:37 — message 工具反复失败最终超时

3. **healthcheck 流程未闭环**
   - 用户选了"1"同意安全扫描 → Jonathan 给出选项菜单
   - 用户后续发新任务 → Jonathan 完全放弃 healthcheck，未提醒有未完成修复项

4. **workspace git 不完整**
   - AGENTS.md、BOOTSTRAP.md、HEARTBEAT.md、SOUL.md、TOOLS.md 仍为 untracked

---

## Memory Behavior

- Day 1：完全被动（需直接提醒），MEMORY.md 未创建
- Day 2：仍需提醒但提炼质量显著提高，MEMORY.md 已建立（4 条原则），能区分短期/长期
- 持续问题：不会自发提炼，TOOLS.md 始终未更新（2/26 壮爸直接修改了 TOOLS.md）

---

## Capability Boundaries（更新）

- **Can**: 搭建完整的运维监控系统（脚本 + 告警 + cron + git）
- **Can**: 创建 GitHub 仓库并推送（需 PAT token）
- **Can**: 准确解释自身架构（skill/agent/memory/context）
- **Cannot**: 面对工具 API 变更自主适配参数（message tool bug）
- **Cannot**: 主动闭环被中断的任务流程
- **Cannot**: 主动写入长期记忆（需用户提醒）
- **Limitation**: 语音消息下载失败（Telegram 媒体获取需代理配置）

---

## Open Issues

| # | 问题 | 发现日期 | 状态 |
|---|------|----------|------|
| 1 | Telegram Bot token 需轮换 | 2/24 | 未处理 |
| 2 | 语音消息 Failed to download media | 2/25 | 方案已提出，用户未确认 |
| 3 | message 工具 `to`/`channelId` vs `target` 兼容问题 | 2/25 | ✅ 已在 TOOLS.md 记录正确参数（2/26 壮爸直接修改） |
| 4 | workspace 多个核心文件 untracked | 2/25 | 未处理 |
| 5 | SSH 绑定 0.0.0.0 + 无防火墙 | 2/24 | healthcheck 发现，未修复 |
| 6 | browser 工具超时导致 Gateway 崩溃重启 | 2/26 | 新发现（访问 Proton Mail 时触发） |

## Recommendations

1. ~~修复 message 工具参数问题~~ ✅ 已在 TOOLS.md 记录（2/26）
2. **配置 Telegram 通道代理** — 解决语音/媒体下载失败
3. **轮换 Bot token** — 已知泄露，优先级高
4. **设置 TOOLS.md 自动检查提醒** — 每次会话结束前提示 Jonathan 检查是否需更新
5. **建立"中断恢复"机制** — 被新任务打断时，先列出未完成项再切换

## Eval Watermark

| 字段 | 值 |
|------|-----|
| last_eval_date | 2026-02-25 |
| session_id | 576c53d4-826d-43aa-8127-3c9d25d4523e |
| last_jsonl_timestamp | 2026-02-25T19:35:00+08:00 |
| journalctl_until | 2026-02-25T19:35:00+08:00 |
