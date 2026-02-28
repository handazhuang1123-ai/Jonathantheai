---
title: Jonathan Agent Swarm 深度分析与实施方案
date: 2026-02-26
author: Claude Opus 4.6 (assisting zhuangba)
tags: [agent-swarm, architecture, planning, openclaw, multi-agent]
depends_on: [2026-02-24_setup-summary.md, 2026-02-24_server-env.md, 2026-02-25_evaluation.md, 2026-02-25_token-analysis.md]
status: draft
last_updated: 2026-02-26
---

# Jonathan Agent Swarm 深度分析与实施方案

> 将 Jonathan 从单兵作战的 AI agent 升级为管理一支 coding agent 编队的编排者。

---

## 目录

1. [执行摘要](#1-执行摘要)
2. [原始架构分析：Elvis 的 Zoe 系统](#2-原始架构分析elvis-的-zoe-系统)
3. [Jonathan 当前状态与约束](#3-jonathan-当前状态与约束)
4. [差距分析](#4-差距分析)
5. [两条技术路线](#5-两条技术路线)
6. [推荐架构：混合顺序执行](#6-推荐架构混合顺序执行)
7. [分阶段实施计划](#7-分阶段实施计划)
8. [资源预算](#8-资源预算)
9. [风险评估与缓解](#9-风险评估与缓解)
10. [成本估算](#10-成本估算)
11. [成功指标](#11-成功指标)
12. [壮爸决策点](#12-壮爸决策点)
13. [参考资料](#13-参考资料)

---

## 1. 执行摘要

### 这是什么

Elvis（推特用户 @elvissun）展示了一种用 OpenClaw 作为编排层、管理多个 Codex/Claude Code coding agent 并行开发的架构。他的 OpenClaw agent "Zoe" 负责理解业务上下文、拆分任务、生成 prompt、派发给 worker agent、监控进度、自动 review，最终通知人类 merge PR。结果是一个人做到了一个开发团队的产出：**日均 50 commits，单日最高 94 commits，30 分钟内 7 个 PR**。

### 我们要做什么

把这个架构适配到 Jonathan 的服务器上。但 Jonathan 的硬件（7.6GB RAM, 4 核 CPU）与 Elvis 的 Mac Mini（16GB）差距巨大——Elvis 自己都觉得 16GB 不够用，花 $3,500 买了 128GB Mac Studio。所以我们不能照搬，必须做深度适配。

### 核心结论（先说结论）

| 维度 | 结论 |
|------|------|
| **可行性** | 可行，但必须严格顺序执行（一次一个 worker agent） |
| **推荐路线** | 混合模式：OpenClaw 原生 subagent 做轻量任务 + tmux 外挂 Claude Code headless 做重度编码 |
| **并行能力** | 当前硬件：最多 1 个 worker agent；升级到 16GB：可尝试 2 个 |
| **预期产出** | 不追求日均 50 commits，但能实现 Jonathan 自主完成"接收任务 → 编码 → 测试 → PR"全流程 |
| **关键前置** | 扩 swap 到 8GB + 安装 zram、安装 pnpm、升级 Claude Code、编写看门狗脚本 |
| **投资** | 纯软件方案 $0 硬件成本；API 费用看模型选择（gpt-5.3-codex-high 在 right.codes 上标价 $0） |

---

## 2. 原始架构分析（Elvis 的 Zoe 系统）

### 2.1 架构概览

```
┌─────────────────────────────────────────────────┐
│                  Elvis (人类)                      │
│         (5-10 分钟 review, merge PR)              │
└─────────────────────┬───────────────────────────┘
                      │ Telegram 通知
┌─────────────────────▼───────────────────────────┐
│            Zoe (OpenClaw Orchestrator)            │
│                                                   │
│  - 持有完整业务上下文 (Obsidian vault)              │
│  - 拆分任务、选择 agent、生成 prompt               │
│  - 监控 agent 进度 (cron 每 10 分钟)               │
│  - 失败时带上下文重试                              │
│  - 主动扫描 Sentry/会议纪要/git log 找活干         │
└──────┬────────────┬──────────────┬──────────────┘
       │            │              │
  ┌────▼────┐  ┌───▼────┐  ┌─────▼─────┐
  │  Codex  │  │ Claude │  │  Gemini   │
  │ Worker  │  │ Code   │  │ (设计)    │
  │ (tmux)  │  │ Worker │  │           │
  │         │  │ (tmux) │  │           │
  └────┬────┘  └───┬────┘  └─────┬─────┘
       │           │              │
  git worktree  git worktree  HTML/CSS spec
       │           │              │
       └───────────┴──────────────┘
               ↓
         GitHub PR → 3 AI Review → CI → Merge
```

### 2.2 八步工作流详解

| 步骤 | 内容 | 关键实现 |
|------|------|----------|
| 1. 需求范围确定 | Zoe 从 Obsidian vault 拉取会议记录，与 Elvis 讨论范围 | 业务上下文驱动 |
| 2. 生成 Agent | 创建 git worktree + tmux session + 运行 codex/claude | `tmux new-session -d` |
| 3. 循环监控 | 10 分钟 cron 检查 tmux/PR/CI，非 agent 轮询 | 确定性脚本，零 token |
| 4. Agent 创建 PR | `gh pr create --fill` | 不通知人类——PR ≠ 完成 |
| 5. 自动代码审查 | 3 个模型审查（Codex/Gemini/Claude） | 捕获不同类型问题 |
| 6. 自动测试 | CI: lint + unit + E2E + Playwright | 全通过才算 done |
| 7. 人工审查 | Telegram 通知，5-10 分钟审查 | 截图 > 读代码 |
| 8. 合并 + 清理 | merge + cron 清理 worktree | daily cron |

### 2.3 Ralph Loop V2（智能重试）

Elvis 方案的精髓不是并行——**而是失败后的智能重试**：

- Agent 失败 → Zoe **不是简单重跑同一个 prompt**
- Zoe 分析失败原因，结合业务上下文**重写 prompt**
- "Agent 上下文爆了？" → "只关注这三个文件。"
- "Agent 方向跑偏了？" → "停。客户要的是 X 不是 Y。"
- "Agent 需要澄清？" → "这是客户的邮件和公司介绍。"

> 这恰恰针对 Jonathan Day 2 的核心缺陷——**40 次用相同参数暴力重试 message 工具**。Ralph Loop V2 是治这个病的药。

### 2.4 为什么有效：上下文分离

Elvis 系统成功的根本原因：

> "Context windows are zero-sum. Fill it with code → no room for business context. Fill it with customer history → no room for the codebase."

- **Zoe（编排层）**：持有业务上下文，生成精准 prompt
- **Worker agents（执行层）**：只看代码，不被业务噪音干扰

**这个洞察与 Jonathan 的 RAM 无关，7.6GB 也能实现上下文分离。**

### 2.5 Elvis 的成本与瓶颈

- Claude API: ~$100/月，Codex API: ~$90/月
- **瓶颈是 RAM**：16GB Mac Mini 极限 4-5 个并行 agent → 花 $3,500 买 128GB Mac Studio

---

## 3. Jonathan 当前状态与约束

### 3.1 硬件实测（2026-02-26 08:00 UTC+8）

| 资源 | 规格 | 实测值 |
|------|------|--------|
| CPU | i5-7300HQ 4C/4T @ 2.50GHz | load average: 0.05（几乎空闲） |
| RAM 总量 | 7.6 GB | - |
| RAM 已用 | 1.4 GB | OpenClaw 400MB + VNC/XFCE ~500MB + 系统 ~500MB |
| RAM 可用 | 6.2 GB（含 buffer/cache） | 充裕 |
| Swap | 4 GB (NVMe) | 仅用 256 KB |
| Disk | 116 GB NVMe | 97 GB 空闲（84%） |
| 运行时间 | 2 天 21 小时 | 稳定 |

### 3.2 当前进程内存 Top 5

| 进程 | RSS (MB) | 说明 |
|------|----------|------|
| openclaw-gateway | ~400 | Jonathan 的大脑 |
| TigerVNC :1 | ~163 | GUI 远程桌面 |
| xfwm4 | ~110 | 窗口管理器 |
| xfce4-session | ~78 | 桌面会话 |
| xfdesktop | ~71 | 桌面 |

> VNC + XFCE 全家桶合计约 **400-500 MB**。如果壮爸已完全通过 SSH + Telegram 管理，关闭 VNC 可立即释放 ~400 MB。

### 3.3 已安装工具清单

| 工具 | 版本 | 状态 | 备注 |
|------|------|------|------|
| OpenClaw | v2026.2.23 | ✅ 运行中 | Gateway 端口 18789 |
| Claude Code CLI | v2.1.42 | ✅ 已安装 | `/usr/bin/claude` |
| Codex CLI | - | ❌ 未安装 | 需要 OpenAI API key |
| tmux | 3.4 | ✅ 已安装 | - |
| git | 2.43.0 | ✅ 已安装 | - |
| Node.js | v22.22.0 | ✅ 已安装 | - |
| pnpm | - | ❌ 未安装 | 一条命令安装 |
| gh (GitHub CLI) | - | 需确认 | 可能已有 |
| jq | - | 需确认 | JSON 处理 |

### 3.4 OpenClaw 关键配置

```json
{
  "maxConcurrent": 4,           // agent 并发上限
  "subagents": {
    "maxConcurrent": 8          // subagent 并发上限
  },
  "contextPruning": {
    "mode": "cache-ttl", "ttl": "1h"
  },
  "heartbeat": { "every": "30m" }
}
```

**关键发现**：OpenClaw 已原生支持 `sessions_spawn` 工具（非阻塞派发子 agent），配置允许最多 8 个并发 subagent。

### 3.5 Jonathan 能力基线（Day 2 评估）

| 维度 | 得分 | 与 Swarm 的关联 |
|------|------|-----------------|
| 指令遵循 | 8/10 | 编排核心能力 ✅ |
| 工具使用 | 7/10 | **40 次暴力重试是隐患** ⚠️ |
| 自我调试 | 6/10 | Ralph Loop V2 可以弥补 |
| 主动记忆 | 6/10 | 编排经验需要记录 |
| Git 工作流 | 7/10 | worktree + PR ✅ |
| 任务规划 | 7/10 | 任务拆分基础 ✅ |
| **综合** | **7/10** | **够用，但需要针对性加固** |

### 3.6 Claude Code CLI 内存风险（关键数据）

| 场景 | RSS 内存 | 来源 |
|------|----------|------|
| Headless 模式 (`claude -p`) | ~256 MB | 社区实测 |
| 交互模式空闲 | ~400-556 MB | GitHub issue |
| 典型活跃使用 | 1-2 GB | 官方预期 |
| v2.1.54+ 内存泄漏回归 | ~1 GB/min 增长 | GitHub #28763 |
| 泄漏峰值 | 8-15+ GB → OOM kill | GitHub #4953, #22188 |

**Jonathan 当前 v2.1.42 在泄漏回归版本（v2.1.54+）之前，但仍需谨慎。**

---

## 4. 差距分析

### 4.1 全维度对比

| 维度 | Elvis (Zoe) | Jonathan | 差距评级 |
|------|-------------|----------|----------|
| RAM | 16 GB → 128 GB | 7.6 GB | 🔴 巨大 |
| CPU | Apple Silicon M2/M4 | i5-7300HQ (2017) | 🟡 明显 |
| 并行 agent | 4-5 → 20+ | **最多 1** | 🔴 策略性 |
| 目标代码库 | 生产 SaaS (收入中) | **待定** | 🟡 需决定 |
| Codex CLI | 已安装，主力 90% | 未安装 | 🟢 可安装 |
| Claude Code | 已安装 | v2.1.42 已安装 | 🟢 已有 |
| pnpm | 使用中 | 未安装 | 🟢 可安装 |
| 编排 skill | 成熟 (coding-agent 等) | 基础配置 | 🟡 需构建 |
| 业务上下文 | Obsidian vault (丰富) | MEMORY.md (4 条原则) | 🟡 需丰富 |
| 网络 | 美国直连 | 中国大陆需代理 | 🟡 额外配置 |
| 安全 | 未提及 | Bot token 泄露 + SSH 暴露 | 🔴 必须先修 |

### 4.2 不可照搬的部分

1. **并行执行** — 5 个并行 agent × 2 GB = 10 GB，直接 OOM
2. **3 个 AI reviewer** — 3 个不同 API provider 太复杂
3. **Sentry 集成** — Jonathan 没有生产系统可监控（暂时）
4. **Obsidian vault** — Jonathan 的记忆系统更轻量

### 4.3 可以直接采用的部分

1. ✅ **上下文分离** — Jonathan 持有高层上下文，worker 只看代码
2. ✅ **tmux + worktree** — 技术零障碍，tmux 3.4 和 git 2.43 已就绪
3. ✅ **任务注册表** — JSON 文件追踪状态
4. ✅ **cron 监控** — 确定性检查，零 token 消耗
5. ✅ **"Done" 定义** — PR + CI 全通过才通知人类
6. ✅ **Ralph Loop V2** — 分析失败原因调整 prompt 重试

---

## 5. 两条技术路线

### 路线 A：OpenClaw 原生 Subagent

利用 `sessions_spawn` 工具在 Gateway 进程内派发子任务。

```
Jonathan (main agent)
    ├── sessions_spawn → Subagent 1 (代码审查)
    │                     └── 使用同一个 LLM API
    │                     └── 独立 session context
    │                     └── 受限工具集
    └── sessions_spawn → Subagent 2 (文档生成)
```

| 优点 | 缺点 |
|------|------|
| 零安装，OpenClaw 原生支持 | 使用同一个 LLM provider |
| 共享 Gateway 进程，内存开销低 | 没有 tmux TTY，不能运行交互式 CLI |
| 内建通信机制 | 不能本地编译/测试 |
| 可配 `runTimeoutSeconds` | 长时间运行增加 Gateway 内存压力 |

**适用**：轻量任务（审查、文档、prompt 起草、信息检索）

### 路线 B：外挂 Claude Code / Codex CLI via tmux

用 shell 工具在 tmux 中启动独立 CLI 进程。

```
Jonathan (OpenClaw agent)
    ├── bash: tmux new-session -d -s "worker-1"
    │         tmux send-keys "claude -p '...' --max-turns 10" Enter
    ├── bash: tmux capture-pane -t worker-1 -p  (监控)
    └── bash: gh pr create --fill  (PR 创建)
```

| 优点 | 缺点 |
|------|------|
| Claude Code 有完整文件编辑/shell/git 能力 | 需要额外 API key 和代理配置 |
| 独立进程，crash 不影响 Gateway | Claude Code 有内存泄漏风险 |
| Headless `-p` 模式仅 ~256 MB | 进程间通信靠文件/tmux 输出 |
| 可运行编译/测试/lint | `--dangerously-skip-permissions` 有安全风险 |
| 可使用不同 API provider | - |

**适用**：重度编码（写代码、调试、重构、创建 PR）

### 路线选择：混合模式（推荐）

**不是二选一，而是组合使用。**

| 任务类型 | 使用路线 | 理由 |
|----------|----------|------|
| 代码审查、文档、prompt 起草 | A (原生 subagent) | 轻量、不需要本地执行 |
| 编写代码、运行测试 | B (Claude Code headless) | 需要文件编辑和 shell |
| 创建 PR、git 操作 | B (Claude Code headless) | 需要 git push + gh |

---

## 6. 推荐架构：混合顺序执行

### 6.1 整体架构

```
┌─────────────────────────────────────────────────────────┐
│                    壮爸 (Telegram)                         │
│          "帮我做一个 XX 功能" / review PR / merge          │
└───────────────────────┬─────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────┐
│              Jonathan (OpenClaw Orchestrator)              │
│                                                           │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────────┐  │
│  │ 业务上下文    │ │ 任务队列管理  │ │ Agent 生命周期    │  │
│  │ (memory/)    │ │ (tasks.json) │ │ (spawn/monitor/  │  │
│  │              │ │              │ │  kill/respawn)   │  │
│  └──────────────┘ └──────────────┘ └──────────────────┘  │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────────┐  │
│  │ Prompt 工程   │ │ 进度通知     │ │ 看门狗 + 资源监控 │  │
│  └──────────────┘ └──────────────┘ └──────────────────┘  │
└───────────────────────┬─────────────────────────────────┘
                        │ shell / tmux / git
                        ▼
┌─────────────────────────────────────────────────────────┐
│  编码 Agent 层 (tmux sessions) — 严格一次一个              │
│                                                          │
│  ┌─────────────────────────────────────────┐             │
│  │ tmux: worker-{task-id}                   │             │
│  │ claude -p "..." --max-turns N            │             │
│  │ 工作目录: git worktree (feat/xxx)        │             │
│  │ 超时: 30 min                             │             │
│  │ 看门狗: RSS > 2GB → kill                 │             │
│  └─────────────────────────────────────────┘             │
│                                                          │
│  ┌─────────────────────────────────────────┐             │
│  │ memory-guard (cron 每分钟)               │             │
│  │ 可用 RAM < 1.5GB → 暂停 / 可用 < 800MB → kill        │
│  └─────────────────────────────────────────┘             │
└───────────────────────┬─────────────────────────────────┘
                        │ git push + gh pr create
                        ▼
┌─────────────────────────────────────────────────────────┐
│  GitHub → CI → (optional AI review) → 通知壮爸 → Merge   │
└─────────────────────────────────────────────────────────┘
```

### 6.2 任务生命周期

```
1. 壮爸 → Telegram: "给 monitor 系统加磁盘预警"
        ↓
2. Jonathan 接收 → 拆分子任务
   - 子任务 1: 修改 health-check.sh
   - 子任务 2: 修改 alert.sh
   - 子任务 3: 更新 README
        ↓
3. Jonathan 为子任务 1 准备 worker
   a. git worktree add ../monitor-disk -b feat/disk-alert
   b. 生成精准 prompt（含上下文、文件路径、验收标准）
   c. tmux new-session -d -s worker-disk
   d. claude -p "..." --max-turns 15 --output-format json
        ↓
4. Jonathan 监控（每 5 分钟）
   - tmux 存活？ commit 了？ 内存正常？
        ↓
5a. Worker 成功 → commit → push → PR → 通知壮爸
5b. Worker 失败 → Jonathan 分析错误日志 → 调整 prompt → 重试（≤3 次）
        ↓
6. 子任务 1 完成 → 释放 worker → 启动子任务 2（严格顺序）
        ↓
7. 全部完成 → Jonathan 汇总 → Telegram 通知壮爸
```

### 6.3 为什么顺序执行是正确选择（不是妥协）

1. **质量 > 数量** — 一次做好一件事，比同时做三件半成品强
2. **避免 OOM** — 并行 agent 是 7.6GB 机器的死亡螺旋
3. **简化调试** — 出问题时只有一个嫌疑人
4. **Cache 友好** — 顺序执行可以复用 OpenClaw 的 cache（Day 2 已证明 91% 命中率）
5. **Elvis 的瓶颈也是 RAM** — 16GB 只能跑 4-5 个，7.6GB 跑 1 个是等比缩放
6. **核心价值不在并行** — 价值在于 Jonathan 能自主完成"任务→代码→PR"全流程，这不需要并行

### 6.4 与 Elvis 架构的关键差异

| 原始 (Elvis/Zoe) | 适配 (Jonathan) | 原因 |
|-------------------|-----------------|------|
| 4-5 并行 agent | 1 顺序 agent | RAM |
| Codex = 主力 (90%) | Claude Code = 主力 | 已安装 |
| 3 AI reviewer | 1 AI review (可选) | 简化 |
| Obsidian vault | MEMORY.md + workspace | 已有 |
| 无内存管理 | memory-guard.sh + 看门狗 | **必须有** |
| $190/月 API | $0-50/月 | 成本优势 |
| Mac Mini → Mac Studio | 现有服务器不变 | $0 硬件 |

---

## 7. 分阶段实施计划

### Phase 0: 基础设施加固（预计 ~2 小时）

**目标**：让服务器具备承载额外 agent 进程的能力。

| # | 任务 | 操作 | 风险 |
|---|------|------|------|
| 0.1 | 扩 swap 到 8 GB | `fallocate -l 8G` + `mkswap` + `swapon` | 低 |
| 0.2 | 安装 zram 压缩内存 | `apt install zram-tools`（等效增加 ~2-3 GB） | 低 |
| 0.3 | 设置 swappiness=15 | `sysctl vm.swappiness=15` | 低 |
| 0.4 | 升级 Claude Code CLI | `npm update -g @anthropic-ai/claude-code` | 中 |
| 0.5 | 安装 pnpm | `npm install -g pnpm` | 低 |
| 0.6 | 确认/安装 gh, jq | `apt install gh jq` | 低 |
| 0.7 | **[决策] 是否关闭 VNC/XFCE** | 释放 ~400 MB RAM | 看壮爸需求 |
| 0.8 | **[安全] 轮换 Bot token** | Telegram BotFather | 高优先 |
| 0.9 | **[安全] 配置 UFW 防火墙** | `ufw allow ssh && ufw enable` | 高优先 |

> **关于 0.8/0.9**：Agent Swarm 给了 Jonathan 更大的 shell 执行权限。如果 Bot token 泄露导致攻击者控制 Jonathan，而 SSH 暴露无防火墙，攻击者可获得完整 shell 访问。**必须在 Phase 1 之前完成安全加固。**

### Phase 1: 单 Worker 端到端验证（预计 ~3 小时）

**目标**：验证 Jonathan 能成功启动一个 Claude Code worker、完成简单任务、创建 PR。

| # | 任务 | 验收标准 |
|---|------|----------|
| 1.1 | 配置 Claude Code API key + 代理 | `claude -p "echo hello"` 返回结果 |
| 1.2 | 安装 coding-agent skill | `npx playbooks add skill openclaw/skills --skill coding-agent` |
| 1.3 | 安装 tmux-agents skill | `npx playbooks add skill openclaw/skills --skill tmux-agents` |
| 1.4 | 创建 `~/.openclaw/swarm/` 目录结构 | 目录 + tasks.json 模板 |
| 1.5 | 编写看门狗脚本 | 监控 Claude Code RSS，超 2 GB 自动 kill |
| 1.6 | **端到端测试** | 见下方测试场景 |

**Phase 1 端到端测试场景（用 monitor 仓库）：**

```
壮爸 → Jonathan (Telegram):
  "给 monitor 的 health-check.sh 添加网络延迟检测功能，
   用 ping 测试到 8.8.8.8 和 baidu.com 的延迟。"

Jonathan 应该做：
1. git worktree add ../monitor-netcheck -b feat/network-latency
2. 生成 prompt（含 health-check.sh 现有结构、输出格式要求等）
3. tmux new-session -d -s worker-netcheck
4. tmux send-keys "claude -p '...' --max-turns 10" Enter
5. 每 5 分钟检查 tmux 是否存活 + 是否有 commit
6. Worker 完成后: push + gh pr create
7. 通过 Telegram 通知壮爸 PR 链接

成功标准：
- PR 被创建
- health-check.sh 中有网络延迟检测逻辑
- 脚本可运行，输出格式正确
- 全过程无 OOM
- worker 内存未超过 2 GB
```

### Phase 2: 监控与恢复（预计 ~3 小时）

**目标**：系统能自我监控，失败时智能恢复。

| # | 任务 | 验收标准 |
|---|------|----------|
| 2.1 | 编写 memory-guard.sh | RAM < 1.5GB 告警，< 800MB 杀 agent |
| 2.2 | 编写 check-agents.sh | 每 10 分钟检查 tmux/PR/CI 状态 |
| 2.3 | 配置 cron | memory-guard 每分钟，check-agents 每 10 分钟 |
| 2.4 | 实现 agent kill + cleanup | 优雅终止 + 清理 tmux + worktree |
| 2.5 | 实现 Ralph Loop V2 重试 | 失败 → 分析日志 → 调整 prompt → 重试（≤3 次） |
| 2.6 | 集成到 HEARTBEAT.md | Jonathan 心跳巡检包含 swarm 状态 |
| 2.7 | 24 小时稳定性测试 | 服务器运行 24h 无 OOM |

### Phase 3: 智能编排能力（预计 ~4 小时）

**目标**：Jonathan 成为真正的编排者。

| # | 任务 | 验收标准 |
|---|------|----------|
| 3.1 | 任务拆分 prompt 模板 | Jonathan 能把大任务拆成 2-3 个子任务 |
| 3.2 | Worker prompt 模板库 | 标准化结构（背景→目标→约束→文件→验收标准） |
| 3.3 | 顺序调度逻辑 | 任务 1 完成 → 自动启动任务 2 |
| 3.4 | 进度通知 | 通过 Telegram 实时报告进度 |
| 3.5 | 编排经验记录 | 成功/失败模式写入 MEMORY.md |
| 3.6 | 简单任务一次成功率 > 70% | 统计验证 |

**Worker Prompt 模板（草案）：**

```markdown
## 任务
{task_description}

## 背景
{context_from_jonathan_memory}

## 工作目录
{worktree_path}

## 关键文件
{file_list_with_brief_description}

## 约束
- 只修改上述文件范围内的代码
- 所有变更必须通过 lint 和现有测试
- commit message 格式: "feat|fix|refactor: {description}"
- 完成后运行 `git push -u origin {branch_name}`

## 验收标准
{acceptance_criteria}
```

### Phase 4: 扩展（条件触发）

| 触发条件 | 扩展方向 |
|----------|----------|
| RAM 升级到 16GB+ | 尝试 2 个并行 worker |
| 有稳定的软件项目 | 安装 Codex CLI，Codex vs Claude 对比 |
| CI/CD 流水线建好 | 自动化测试 + PR review |
| 需要 UI 设计 | 引入 Gemini 设计 spec → Claude 实现 |
| 项目增多 | WSL 溢出 / 云 VPS 分布式方案 |

---

## 8. 资源预算

### 8.1 Phase 1 后预期内存分布

**场景 A：保留 VNC（保守估算）**

```
组件                        RSS (MB)
────────────────────────────────────────
Linux 内核 + 系统            800
mihomo                       50
OpenClaw Gateway             400-800
VNC + XFCE                   400
Claude Code headless worker  256-512
tmux                         5
────────────────────────────────────────
总计                         1,911 - 2,567
可用 (7,600 - 总计)          5,033 - 5,689
安全余量                     66% - 75%   ✅
```

**场景 B：关闭 VNC**

```
总计                         1,511 - 2,167
可用                         5,433 - 6,089
安全余量                     72% - 80%   ✅✅
```

**场景 C：最坏情况（内存泄漏）**

```
Gateway 膨胀到               2,000
Claude Code 泄漏到           2,000
+ 系统                        850
────────────────────────────────────────
总计                         4,850
可用 RAM                     2,750
+ 8 GB swap (NVMe)           10,750
────────────────────────────────────────
结论：有 swap + zram 兜底，不会 OOM kill
看门狗在 2 GB 时 kill Claude Code
Gateway 在 2 GB 时 /reset
```

### 8.2 磁盘预算

```
当前已用                      11 GB
pnpm global store            2-5 GB（按项目规模）
每个 git worktree (pnpm)     ~500 MB
同时 5 个 worktree           2.5 GB
Swap 文件 (8 GB)             8 GB
预留系统 + 日志              10 GB
────────────────────────────────────────
总计                         ~36 GB
剩余 (116 - 36)             ~80 GB   ✅ 充足
```

---

## 9. 风险评估与缓解

### 🔴 高风险

| 风险 | 影响 | 缓解 |
|------|------|------|
| Claude Code 内存泄漏 → OOM | Gateway 被杀，Jonathan 离线 | 看门狗(2GB kill) + 8GB swap + zram + headless + timeout |
| Bot token 已泄露 + SSH 暴露 | 攻击者获得 shell 权限 | **Phase 0 必须轮换 token + 配 UFW** |
| Jonathan 暴力重试（已知行为） | Token 浪费 + 时间浪费 | 最多 3 次重试 + 重试前**必须修改 prompt** |

### 🟡 中风险

| 风险 | 影响 | 缓解 |
|------|------|------|
| Worker 写坏代码/删文件 | 代码库损坏 | git worktree 隔离 + `--max-turns` 限制 |
| Claude Code API key 管理 | 泄露或配额耗尽 | 环境变量，不写入配置文件 |
| Worker prompt 质量差 | 任务失败率高 | 从简单任务迭代，建立 prompt 库 |
| 代理/网络问题 | Worker 无法访问 API | 继承 mihomo 代理配置 |

### 🟢 低风险

| 风险 | 缓解 |
|------|------|
| tmux session 意外终止 | tasks.json 记录状态，可恢复 |
| 磁盘空间不足 | 97 GB 余量 + pnpm + 自动清理 |
| Git worktree 残留 | daily cron 清理 orphan |

### 安全策略：`--dangerously-skip-permissions`

Elvis 使用这个标志实现全自动。但它意味着 Claude Code 可以不经确认：删文件、执行任意命令、push 代码、装卸包。

**推荐分阶段放权**：
- Phase 1-2：**不使用**，让 Jonathan 手动确认敏感操作
- Phase 3：用 allowlist 机制（`settings.json` 配置允许的命令模式）替代全跳过
- Phase 4+：评估后决定是否完全自动

---

## 10. 成本估算

### 10.1 API 成本

| 提供商 | 模型 | 用途 | 估算月费 |
|--------|------|------|----------|
| right.codes | gpt-5.3-codex-high | Jonathan 编排层 | 已有（按现有安排） |
| Anthropic | Claude (via CLI) | Worker agent | $20-100/月 |
| OpenAI | Codex (via CLI) | Worker agent（可选） | $20-90/月 |

**最低成本方案**：如果 Claude Code CLI 或 Codex CLI 能配置使用 right.codes endpoint（OpenAI 兼容格式），worker 也可零额外 API 费用。需验证。

### 10.2 硬件成本

| 选项 | 成本 | 收益 |
|------|------|------|
| 现状不变 | $0 | 顺序 1 worker |
| 加 8GB DDR4 SODIMM | ~¥100-150 | 可能 2 并行（需确认槽位） |
| 换更高配服务器 | 视配置 | 更多并行 + 更快编译 |

### 10.3 推荐成本策略

**起步（Phase 1-2）**：
- Jonathan 编排层继续用 right.codes
- Worker **只用 Claude Code CLI**（按量付费，可控）
- 不引入 Codex（避免额外固定成本）
- 预估月增量：**$20-50**

**进阶（Phase 3+）**：
- 简单任务路由到 claude-haiku-4-5（便宜 90%+）
- 复杂任务用 claude-opus-4.5/4.6
- 预估月增量：**$50-100**

---

## 11. 成功指标

### Phase 1 = 以下全部达成

- [ ] Jonathan 能通过 Telegram 接收编码任务
- [ ] Jonathan 能创建 git worktree + 启动 Claude Code worker
- [ ] Worker 能完成简单编码（修改 1-2 个文件）
- [ ] Worker 完成后自动 commit + push + PR
- [ ] Jonathan 通过 Telegram 通知壮爸 PR 链接
- [ ] 全过程无 OOM，worker 内存未超 2 GB

### Phase 2 = 以上 + 以下全部达成

- [ ] 服务器运行 24h 无 OOM
- [ ] 失败任务自动重试（调整 prompt，非盲重）
- [ ] memory-guard 正常运行

### Phase 3 = 以上 + 以下全部达成

- [ ] Jonathan 能拆分中等复杂度任务为 2-3 个子任务
- [ ] 顺序调度多个子任务
- [ ] 简单任务一次成功率 > 70%
- [ ] "任务 → PR 可 merge"平均 < 1 小时
- [ ] Jonathan 主动记录编排经验到 MEMORY.md

### 终极目标

> 壮爸在手机上说"做 X"，然后去忙别的。回来时看到 Telegram："PR 已创建，CI 通过，等待 review。"

---

## 12. 壮爸决策点

以下决策直接影响实施方案，需要你拍板：

### D1: VNC/XFCE 是否可关？

- 关闭释放 ~400 MB RAM（在 7.6GB 机器上很可观）
- 可随时重新启用
- 如果已完全通过 SSH + Telegram 管理，GUI 无用

### D2: Worker agent 用什么 API？

| 选项 | 月费 | 说明 |
|------|------|------|
| A) 让 CLI 也走 right.codes | $0 | 需验证兼容性 |
| B) 购买 Anthropic API credit | $20-100 | Claude 原生体验 |
| C) 购买 OpenAI API（装 Codex） | $20-90 | Codex 重度编码可能更强 |
| D) 两者都买 | $40-190 | 最大灵活性 |

### D3: 第一个项目是什么？

| 选项 | 说明 |
|------|------|
| A) monitor 仓库 | 已有，简单，适合 Phase 1 验证 |
| B) 新建练手项目 | 如简单 web app，让 Jonathan 从零构建 |
| C) 壮爸指定的真实项目 | 最有实际价值 |
| D) 先建基础设施，后定项目 | 灵活 |

### D4: 安全级别

| 选项 | 说明 |
|------|------|
| A) 保守（推荐） | Worker 不跳过权限，Jonathan 确认敏感操作 |
| B) 半自动 | allowlist（允许 git push、文件编辑，禁止 rm/系统命令） |
| C) 全自动 | `--dangerously-skip-permissions` |

### D5: 安全问题先修还是后修？

| 选项 | 说明 |
|------|------|
| A) **先修（推荐）** | Bot token 轮换 + UFW → 再 Phase 1 |
| B) 并行处理 | Phase 1 和安全同步进行 |
| C) 接受风险 | 先跑起来 |

### D6: 月度预算

- A) $20 以内
- B) $50 以内（推荐）
- C) $100 以内
- D) 不限

---

## 13. 参考资料

### 核心

- [Elvis Agent Swarm (X/Twitter)](https://x.com/elvissun/status/2025920521871716562) — 灵感来源
- [OpenClaw Sub-Agents 文档](https://docs.openclaw.ai/tools/subagents) — sessions_spawn 机制
- [OpenClaw Multi-Agent 文档](https://docs.openclaw.ai/concepts/multi-agent) — 多 agent 路由

### OpenClaw Skills（关键依赖）

- [coding-agent](https://playbooks.com/skills/openclaw/skills/coding-agent) — Codex/Claude Code 后台 worker 管理
- [tmux-agents](https://playbooks.com/skills/openclaw/skills/tmux-agents) — tmux 会话管理
- [codex-sub-agents](https://playbooks.com/skills/openclaw/skills/codex-sub-agents) — Codex CLI 集成
- [claude-team](https://playbooks.com/skills/openclaw/skills/claude-team) — Claude Code 多 worker 编排
- [self-improving-agent](https://playbooks.com/skills/openclaw/skills/self-improving-agent) — 自我改进循环

### 技术

- [Claude Code 内存泄漏 #4953](https://github.com/anthropics/claude-code/issues/4953)
- [Claude Code RSS 回归 #28763](https://github.com/anthropics/claude-code/issues/28763)
- [pnpm + Git Worktree](https://akshay-na.medium.com/pnpm-streamlining-javascript-development-in-conjunction-with-git-worktree-8286a046e3c0)
- [Git Worktrees for AI Coding](https://blog.ramit.io/blog/22-02-2026/git-worktrees-for-aiassisted-coding/)
- [workmux](https://github.com/raine/workmux) — Git Worktrees + tmux
- [ccswarm](https://github.com/nwiizo/ccswarm) — Claude Code Agent Swarm
- [agent-deck](https://github.com/asheshgoplani/agent-deck) — Terminal Session Manager
- [LLM Codegen + Worktrees + tmux](https://dev.to/skeptrune/llm-codegen-go-brrr-parallelization-with-git-worktrees-and-tmux-2gop)

### 安全

- [OpenClaw 安全分析](https://github.com/affaan-m/everything-claude-code/blob/main/the-openclaw-guide.md) — 警告 20% marketplace skill 有恶意
- OpenClaw AGENTS.md — "do not create/remove/modify git worktree checkouts unless explicitly requested"

---

## 附录 A: active-tasks.json Schema

```json
{
  "id": "feat-network-latency",
  "description": "给 health-check.sh 添加网络延迟检测",
  "tmuxSession": "worker-netcheck",
  "agent": "claude-code",
  "repo": "Jonathantheai/monitor-2026-02-25",
  "branch": "feat/network-latency",
  "worktree": "../monitor-netcheck",
  "prompt": "（完整 prompt 存于 .clawdbot/prompts/feat-network-latency.md）",
  "status": "running",
  "retryCount": 0,
  "maxRetries": 3,
  "timeoutMinutes": 30,
  "startedAt": 1740000000000,
  "completedAt": null,
  "pr": null,
  "checks": {
    "prCreated": false,
    "ciPassed": false,
    "reviewPassed": false
  },
  "failureLog": [],
  "notifyOnComplete": true
}
```

## 附录 B: 为什么不直接用 Claude Code Swarm Mode

2026 年初 Anthropic 发布了 Claude Code Swarm Mode / Agent Teams。但不适合 Jonathan：

1. **每个 teammate 是独立进程** — 多实例 × 1-2 GB = OOM
2. **需要 Anthropic API 直连** — Jonathan 主模型走 right.codes
3. **没有 OpenClaw 编排层** — Swarm 的 lead 是 Claude Code，不是 OpenClaw
4. **不支持 Codex worker** — 只能 Claude，不能混合

Jonathan 的优势在于 OpenClaw 的灵活性：任意 LLM 编排 + 任意 CLI agent 执行。

## 附录 C: 关键命令速查

```bash
# === 基础设施 ===
sudo fallocate -l 8G /swap.img && sudo mkswap /swap.img && sudo swapon /swap.img
sudo apt install zram-tools
echo 'vm.swappiness=15' | sudo tee -a /etc/sysctl.conf && sudo sysctl -p
npm install -g pnpm
npm update -g @anthropic-ai/claude-code

# === Worker 管理 ===
git worktree add ../task-xxx -b feat/task-xxx origin/main
tmux new-session -d -s worker-xxx -c "/path/to/worktree"
tmux send-keys -t worker-xxx "claude -p 'prompt' --max-turns 10" Enter
tmux capture-pane -t worker-xxx -p -S -100       # 读取输出
tmux has-session -t worker-xxx 2>/dev/null        # 检查存活
git worktree remove ../task-xxx                   # 清理

# === 看门狗 ===
ps -o rss= -p $(pgrep -f "claude") 2>/dev/null   # 查 RSS (KB)
pkill -f "claude" --signal TERM                   # 优雅终止
```

---

*本文档基于 2026-02-26 的调研，数据来源包括 Elvis 的 Agent Swarm 原文、OpenClaw 官方文档、Claude Code/Codex CLI 文档、GitHub issues、Jonathan 服务器实测数据。所有方案经过资源可行性验证，实际效果需通过 Phase 1 端到端测试确认。*
