---
title: OpenClaw Jonathan Setup Summary
date: 2026-02-24
session_time: "17:00 - 22:30 CST"
author: Claude Opus 4.6 (assisting zhuangba)
tags: [openclaw, setup, installation, onboarding]
depends_on: []
status: current
last_updated: 2026-02-27
---

# OpenClaw Setup Summary - 2026-02-24

## What Was Done

### 1. OpenClaw Installation
- **Version**: v2026.2.23
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
- **Provider type**: Custom (OpenAI-compatible endpoint)
- **Endpoint ID**: `custom-right-codes`
- **Base URL**: `https://right.codes/codex/v1`
- **Model**: `gpt-5.3-codex-high`
- **Context window**: Set to 1,000,000 (was initially 4096, too small)
- **Max tokens**: 32,768

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

### 7. Harness (Autonomous Coding Agent)
- **Install path**: `~/projects/harness-openai/`
- **Wrapper script**: `~/projects/harness-openai/run_harness.sh`
- **Operation manual**: `~/.openclaw/workspace/` 下 PLAYBOOK.md（入口）+ APP_SPEC_GUIDE.md + MDIE.md
- **Python venv**: `~/projects/harness-openai/venv/` (aiohttp + playwright)
- **Details**: See `2026-02-26_harness-deployment.md`

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
- Harness 操作手册：`~/.openclaw/workspace/`（PLAYBOOK.md + APP_SPEC_GUIDE.md + MDIE.md）
- Harness 项目产出：`~/projects/{project}/.issue_store/issues.json`

> 注意：局域网 IP 由 DHCP 分配，重启后可能变动。如网络环境变化，需先确认 IP 或改用 Tailscale。

## Access Methods
- **Dashboard (via SSH tunnel)**: `ssh -L 18789:localhost:18789 zhuangba@192.168.0.18` then `http://localhost:18789/#token=...`
- **noVNC (via SSH tunnel)**: `ssh -L 6080:localhost:6080 zhuangba@192.168.0.18` then `http://localhost:6080/vnc.html`
- **Telegram**: Direct message @zhuangba_openclaw_1st_bot
