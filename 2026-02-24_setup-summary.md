---
title: OpenClaw Jonathan Setup Summary
date: 2026-02-24
session_time: "17:00 - 22:30 CST"
author: Claude Opus 4.6 (assisting zhuangba)
tags: [openclaw, setup, installation, onboarding]
depends_on: []
status: current
last_updated: 2026-02-24
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

## Access Methods
- **Dashboard (via SSH tunnel)**: `ssh -L 18789:localhost:18789 zhuangba@192.168.0.18` then `http://localhost:18789/#token=...`
- **noVNC (via SSH tunnel)**: `ssh -L 6080:localhost:6080 zhuangba@192.168.0.18` then `http://localhost:6080/vnc.html`
- **Telegram**: Direct message @zhuangba_openclaw_1st_bot
