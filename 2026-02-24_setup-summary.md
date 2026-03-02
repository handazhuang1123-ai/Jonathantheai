---
title: OpenClaw Jonathan Setup Summary
date: 2026-02-24
session_time: "17:00 - 22:30 CST"
author: Claude Opus 4.6 (assisting zhuangba)
tags: [openclaw, setup, installation, onboarding, oura]
depends_on: []
status: current
last_updated: 2026-03-02
---

# OpenClaw Setup Summary - 2026-02-24

## What Was Done

### 1. OpenClaw Installation
- **Version**: v2026.2.26（3/2 升级，支持 External Secrets Management）
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
- **Model**: `gpt-5.3-codex-high`
- **Reasoning**: true（3/2 开启）
- **Context window**: 1,000,000
- **Max tokens**: 32,768

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

### 4. Channel
- **Platform**: Telegram Bot API
- **Bot name**: @zhuangba_openclaw_1st_bot
- **Display name**: Jonathan
- **Polling mode**: enabled, running
- **Pairing**: Approved for zhuangba's Telegram account (user ID 见 credentials.md)

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
| Workspace | `~/.openclaw/workspace/` |
| Session data | `~/.openclaw/agents/main/sessions/` |
| mihomo config | `/etc/mihomo/config.yaml` |
| mihomo service | `/etc/systemd/system/mihomo.service` |

## Issues Encountered & Resolved
1. **Context window too small (4096)** → Set to 1,000,000 in model config
2. **Telegram API unreachable** → Installed mihomo proxy with subscription
3. **Health check failed on LAN bind** → Changed to loopback, access via SSH tunnel
4. **Git commit failed (no user info)** → Configured git global user
5. **noVNC for remote viewing** → Installed and configured on port 6080
6. **Gateway 自杀事件（3/1）** → right.codes 余额耗尽后，MiniMax 驱动的 Jonathan 建议执行 `openclaw gateway stop`，导致 gateway 停止；旧 session 恢复后继续执行 gateway 命令形成死循环。修复：归档有毒 session + workspace MEMORY.md 第 9 条禁止建议停止自身 gateway。**架构约束**：gateway 重启必须由外部执行（壮爸手动 `systemctl --user restart openclaw-gateway`），Jonathan 无法安全重启自己的 gateway
7. **Secrets 激活（3/2）** → OpenClaw 2026.2.26 External Secrets Management 已启用（`openclaw secrets audit` clean，`openclaw secrets reload` 成功）

### 7. Skills
- **x-tweet-fetcher**: `~/.openclaw/workspace/skills/x-tweet-fetcher` → `~/projects/x-tweet-fetcher`（symlink，auto-discovered as `openclaw-workspace` source）
- **注册方式**: workspace/skills/ 目录下创建 symlink，gateway 自动发现（无需重启）

### 8. Harness (Autonomous Coding Agent)
- **Install path**: `~/projects/harness-openai/`
- **Wrapper script**: `~/projects/harness-openai/run_harness.sh`
- **Operation manual**: `~/.openclaw/workspace/` 下 PLAYBOOK.md（入口）+ APP_SPEC_GUIDE.md + MDIE.md
- **Python venv**: `~/projects/harness-openai/venv/` (aiohttp + playwright)
- **Details**: See `2026-02-26_harness-deployment.md`

### 9. Oura Ring Integration（健康数据）
- **认证**: OAuth2（PAT 已被 Oura 废弃），Client ID `503e9756-b9a7-45c0-9d2e-3b80d8b8a5ad`
- **Token 存储**: `~/.oura/credentials.json`（access_token 30 天有效，脚本自动 refresh）
- **同步脚本**: `~/monitor/oura-sync.py`（cron 每天 08:10）
- **数据存储**: `~/.oura/data/YYYY-MM-DD.json`（含 daily_sleep, daily_readiness, daily_activity, daily_spo2, daily_stress, heartrate, sleep）
- **Telegram 推送**: 通过 `~/monitor/alert.sh` 发送健康日报（睡眠/恢复/活动/血氧/压力 + 异常预警）
- **Jonathan 感知**: workspace `MEMORY.md` 第 8 条，渐进式披露（用户问健康问题时读 ~/.oura/data/）
- **会员要求**: Ring 4 必须有 Oura Membership 才能使用 API

## External Monitoring Interface（壮爸 → Jonathan 的观测接口）

壮爸通过 Claude Code（WSL 环境）SSH 进入 Jonathan 所在服务器来监控和评估 Jonathan。这是获取第一手数据的**唯一可靠接口**。

| 项目 | 值 |
|------|-----|
| **连接命令** | `ssh -i ~/.ssh/id_ed25519 zhuangba@192.168.0.18` |
| **当前网络** | 局域网直连（WSL 和服务器在同一 LAN） |
| **备用连接** | `ssh -i ~/.ssh/id_ed25519 zhuangba@100.79.146.9`（Tailscale，跨网络时用） |
| **可观测数据** | 会话 JSONL、systemd 日志、workspace 文件、实际产出物 |

关键路径：
- 对话记录：`~/.openclaw/agents/main/sessions/*.jsonl`
- 服务日志：`journalctl --user -u openclaw-gateway --since today`
- workspace：`~/.openclaw/workspace/`（含 MEMORY.md、memory/、git 历史）
- 监控报告：`~/monitor/reports/`
- MiniMax 用量日报：`~/monitor/token-usage-report.sh`（cron 每天 08:05，推送 Telegram，含余额查询 `~/monitor/balance_query.py`）
- Harness 操作手册：`~/.openclaw/workspace/`（PLAYBOOK.md + APP_SPEC_GUIDE.md + MDIE.md）
- Harness 项目产出：`~/projects/{project}/.issue_store/issues.json`
- Oura 健康数据：`~/.oura/data/YYYY-MM-DD.json`（cron 每天 08:10 同步）

> 注意：局域网 IP 由 DHCP 分配，重启后可能变动。如网络环境变化，需先确认 IP 或改用 Tailscale。

## Access Methods
- **Dashboard (via SSH tunnel)**: `ssh -L 18789:localhost:18789 zhuangba@192.168.0.18` then `http://localhost:18789/#token=...`
- **noVNC (via SSH tunnel)**: `ssh -L 6080:localhost:6080 zhuangba@192.168.0.18` then `http://localhost:6080/vnc.html`
- **Telegram**: Direct message @zhuangba_openclaw_1st_bot
