---
title: OpenClaw Jonathan Setup Summary
date: 2026-02-24
session_time: "17:00 - 22:30 CST"
author: Claude Opus 4.6 (assisting zhuangba)
tags: [openclaw, setup, installation, onboarding, oura]
depends_on: []
status: current
last_updated: 2026-03-07
---

# OpenClaw Setup Summary - 2026-02-24

## What Was Done

### 1. OpenClaw Installation
- **Version**: v2026.3.2（3/3 升级，installer + mihomo 代理；npm 直连极慢需设 HTTP_PROXY）
- **Install method**: `curl -fsSL https://openclaw.ai/install.sh | bash`
- **Install path**: `/home/zhuangba/.npm-global/bin/openclaw`
- **Node.js**: v22.22.0 (pre-existing, meets Node 22+ requirement)
- **PATH fix**: Added `/home/zhuangba/.npm-global/bin` to `~/.bashrc`

### 2. Onboarding Configuration
- **Mode**: Manual (not QuickStart, to set custom Base URL)
- **Gateway port**: 18789
- **Gateway bind**: Changed to `loopback` (was `LAN`, caused security error)
- **Gateway auth**: Token (auto-generated)
- **Gateway token**: (见 credentials.md)
- **Tailscale exposure**: Off
- **Systemd service**: Enabled with lingering for user `zhuangba`
- **Service file**: `/home/zhuangba/.config/systemd/user/openclaw-gateway.service`
- **Runtime**: Node (recommended)

### 3. Model Provider

**主模型**：
- **Provider type**: Custom (OpenAI-compatible endpoint)
- **Endpoint ID**: `custom-right-codes`
- **Base URL**: `https://right.codes/codex/v1`
- **Model**: `gpt-5.4-high`（3/6 升级，原 gpt-5.3-codex-high）
- **Reasoning**: true（3/2 开启）
- **Fallback 链（3/6）**: gpt-5.4-high → gpt-5.3-codex-high → MiniMax-M2.5（三层）
- **502 fallback 已知限制**: OpenClaw 将 502 归类为 "timeout" → 重试同 provider 不 cross-provider failover。同 provider 的两层都失败时 MiniMax 不会被触发
- **Context window**: 1,000,000
- **Max tokens**: 32,768
- **Embedded run timeout**: `agents.defaults.timeoutSeconds: 1800`（3/7 修复，默认 600s 导致长任务 10 分钟中断。修改 openclaw.json → 重启 gateway 生效）

**Fallback 模型（3/1 新增）**：
- **Provider**: `minimax`
- **API**: `anthropic-messages`（必须用此模式，openai-completions 模式下 thinking 标签会导致空 tool_call_id 400 错误）
- **Base URL**: `https://api.minimax.io/anthropic`（国际站，需走 mihomo 代理）
- **Model**: `MiniMax-M2.5`（旗舰，SWE-Bench 80.2%）
- **Context window**: 1,000,000
- **Max tokens**: 131,072
- **Reasoning**: true（3/2 开启，MiniMax M2.5 本身是推理模型，开启后 extended thinking 效果更好）
- **计费**: 按量付费。JSONL 中不记录 reasoning tokens，本地脚本无法精确估算成本，改为**余额手动汇报制**（壮爸告知余额 → Jonathan 写入 `memory/minimax-balance.json`）
- **Fallback 逻辑**: right.codes 返回错误时自动切换（403/401→"auth" reason、5xx→"timeout" reason 均触发）
- **验证（3/1）**: right.codes 403 余额不足 → cooldown 机制跳过 → MiniMax 接管，Jonathan 正常回复
- **cost 配置**: 已删除（3/2），成本用余额手动汇报制，不依赖 openclaw.json cost 字段

### 4. Channel & Multi-Agent 架构（3/3 新增）

**九 agent 体系**（3/3 搭建，3/6 扩展至 8 个，3/7 新增 urban-regeneration）：

| Agent ID | 名称 | 模型 | Bot | Workspace |
|----------|------|------|-----|-----------|
| main | Jonathan | gpt-5.4-high | @zhuangba_openclaw_1st_bot | `~/.openclaw/workspace/` |
| gatekeeper | 看门老大爷 | gpt-5.4-high | @kanmen_laodaye_bot | `~/.openclaw/workspace-gatekeeper/` |
| nurse | 性感小护士 | gpt-5.3-codex-medium | @sexy_hot_nurse_bot | `~/.openclaw/workspace-nurse/` |
| teacher | 大拿 | gpt-5.4-high | @teacher_dana_bot | `~/.openclaw/workspace-teacher/` |
| mail-assistant | 邮件助理 | gpt-5.4-high | (无独立 bot) | `~/.openclaw/workspace-mail-assistant/` |
| naonao | 脑脑 | gpt-5.4-high | @obsidion_brainstorm_bot | `~/.openclaw/workspace-naonao/` |
| chaige | 拆哥 | gpt-5.4-high | @idontagree_bot | `~/.openclaw/workspace-chaige/` |
| jianjie | 建姐 | gpt-5.4-high | @listen_to_jiejie_bot | `~/.openclaw/workspace-jianjie/` |
| urban-regeneration | 城市更新 | gpt-5.4-high | (无独立 bot) | `~/.openclaw/workspace-urban-regeneration/` |

- **Agent Registry**（3/6 新增）：结构化清单 `~/.openclaw/shared/agent-registry.json`，Jonathan 和脚本均可读取
- **Polling mode**: enabled, running
- **Routing**: bindings 按 `channel + accountId` 路由到对应 agent
- **IDENTITY.md**: 平台层元数据（显示名、头像），不被 LLM 读取
- **SHARED_RULES.md**（3/6 新增）：跨 agent 共享规则，源文件 `~/.openclaw/shared/SHARED_RULES.md`，各 workspace 通过 symlink 引用。git 追踪变更（Jonathan commit）
- **Workspace 标准化（3/6）**：AGENTS.md 精简到 ~40 行，启动序列统一（SOUL→SHARED_RULES→USER→MEMORY）。USER.md 统一称呼"壮爸"。emoji 全部移除
- **agent-create skill**（3/6 新增）：Jonathan workspace/skills/agent-create/SKILL.md，对话式创建/审计/退役 agent。已验证：邮件助理（自动创建）+ 脑脑（对话式创建）
- **Brainstorm Storm（3/6 新增）**：群聊风暴编排器 `~/.openclaw/shared/brainstorm/brainstorm.py`，通过 LLM API 直接驱动三人讨论（Jonathan 主持 + 拆哥 + 建姐），消息发到 Telegram 群 `脑力激荡小组`。Jonathan 通过 groupSystemPrompt 触发执行。拆哥/建姐 groupPolicy=disabled（仅编排器控制发言）
- **群聊配置发现（3/6）**：群聊 session 不注入 startup sequence（不读 SOUL/MEMORY）；groupSystemPrompt 通过 `groups.*.systemPrompt` 注入；bot 互相看不到消息（Telegram Bot API 限制）；`owner` 不是 openclaw.json 合法键
- **Blackboard v1.0**（3/6 新增）：老大爷 weekly-memory-scan.sh 扫描所有 MEMORY.md 增量 → 写 `~/.openclaw/shared/pending-review.md` → Jonathan heartbeat 读取分析 → 推送壮爸确认。cron 每周一 09:00
- **Evolution Protocol（3/6 新增）**：SHARED_RULES 协议14-16（自检信号/弹性容量/错误记录），8 agent 通过 symlink 自动继承。`audit-collector.sh` cron 08:30 零 token 采集 → `shared/daily-audit.txt` → 老大爷 heartbeat 审计
- **环形监督（3/6 落地）**：Jonathan HEARTBEAT.md 新增老大爷履职审计（3 项检查）；全 agent Git 卫生检查 HEARTBEAT section
- **Harness 职能分离（3/6）**：Jonathan 负责需求深挖+写spec+启动，老大爷通过 heartbeat 接管 MDIE 监控循环。MDIE.md 通过 symlink 共享，`shared/harness-active-project.txt` 为项目注册文件
- **gatekeeper 职能**：0-token 状态监控（cron 每 2 分钟）+ 每日质量审计（cron 08:30）+ Harness MDIE 监控（heartbeat 30 分钟）。`workspace-gatekeeper/scripts/monitor/`（collector+evaluator+dispatcher）
- **nurse 健康职能**：Oura 同步 + 健康日报推送已从 main 迁移到 nurse
- **teacher 教学职能**（3/3 新增）：通用教学陪跑教练，边执行边科普，记忆壮爸的学习进度。SOUL.md 定义人格+抽象意图，missions/current.md 定义具体任务（可替换）
- **teacher 当前 mission（3/3 确定）**：MTG Virtual Playtable → 上线。项目源码已部署到 `~/projects/mtg-playtable/`（React+Vite+Express+Socket.IO+Prisma+Neon PostgreSQL+JWT 认证，TypeScript monorepo）。下一步：本地 `npm run dev` 验证 → 容器化 → 云部署 → 域名

### 5. Proxy (mihomo)
- See `2026-02-24_server-env.md` for details
- Required because server in mainland China cannot access Telegram API

### 6. Git & GitHub
- **Git user**: Jonathantheai
- **Email**: Jonathantheai@proton.me
- **GitHub account**: https://github.com/Jonathantheai
- **SSH key**: `/home/zhuangba/.ssh/id_ed25519_github`
- **SSH config**: Configured to use mihomo proxy for github.com
- **Connection verified**: `Hi Jonathantheai! You've successfully authenticated`

## Key Config Files
| File | Path |
|------|------|
| OpenClaw config | `~/.openclaw/openclaw.json` |
| Gateway service | `~/.config/systemd/user/openclaw-gateway.service` |
| Proxy override | `~/.config/systemd/user/openclaw-gateway.service.d/proxy.conf` |
| Workspaces | `~/.openclaw/workspace/`（main）, `~/.openclaw/workspace-{agent}/`（其余 7 个） |
| Shared (rules+registry+brainstorm) | `~/.openclaw/shared/` (git tracked) |
| Session data | `~/.openclaw/agents/{agent}/sessions/*.jsonl` |
| mihomo config | `/etc/mihomo/config.yaml` |
| mihomo service | `/etc/systemd/system/mihomo.service` |

## Issues Encountered & Resolved
1-5: Context window/mihomo/loopback/git user/noVNC（已解决） | 6. Gateway 自杀（3/1，已修复） | 7. Secrets 已启用（3/2）

### 7. Skills
- **x-tweet-fetcher**: `~/.openclaw/workspace/skills/x-tweet-fetcher` → `~/projects/x-tweet-fetcher`（symlink，auto-discovered as `openclaw-workspace` source）
- **注册方式**: workspace/skills/ 目录下创建 symlink，gateway 自动发现（无需重启）

### 8. Harness (Autonomous Coding Agent)
- **Install path**: `~/projects/harness-openai/`
- **Wrapper script**: `~/projects/harness-openai/run_harness.sh`
- **Operation manual**: `~/.openclaw/workspace/` 下 PLAYBOOK.md（入口）+ APP_SPEC_GUIDE.md + MDIE.md
- **Python venv**: `~/projects/harness-openai/venv/` (aiohttp + playwright)
- **Details**: See `2026-02-26_harness-deployment.md`

### 9. Oura Ring Integration（健康数据，3/3 已迁移到 nurse agent）
- **认证**: OAuth2，Token `~/.oura/credentials.json`（30 天有效，自动 refresh）
- **同步**: `~/monitor/oura-sync.py`（cron 08:10），数据 `~/.oura/data/YYYY-MM-DD.json`
- **推送**: 已迁移到 nurse agent（通过 `--account nurse` 发送健康日报）
- **感知**: main workspace MEMORY #8 + nurse workspace 专属职能

## External Monitoring Interface

壮爸通过 Claude Code（WSL）SSH 进入服务器监控 Jonathan。连接信息见 `CLAUDE.local.md`。

关键路径：
- 对话记录：`~/.openclaw/agents/{agent}/sessions/*.jsonl`
- 服务日志：`journalctl --user -u openclaw-gateway --since today`
- workspace：`~/.openclaw/workspace*/`
- 监控报告：`~/monitor/reports/`
- MiniMax 用量日报：`~/monitor/token-usage-report.sh`（cron 08:05）
- Harness 产出：`~/projects/{project}/.issue_store/issues.json`
- Oura 健康数据：`~/.oura/data/YYYY-MM-DD.json`（cron 08:10）
