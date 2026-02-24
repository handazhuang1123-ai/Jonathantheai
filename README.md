---
title: OpenClaw Jonathantheai - Memory Index
created: 2026-02-24
owner: zhuangba (壮爸)
purpose: Context transfer for new Claude sessions about Jonathan (OpenClaw agent)
---

# OpenClaw Jonathantheai - Memory Index

This directory contains structured memory files from sessions managing **Jonathan**, an OpenClaw AI agent running on zhuangba's Linux server.

## Quick Context for New Sessions

**Jonathan** is an OpenClaw v2026.2.23 agent running on a Linux server (i5-7300HQ/8GB/GTX1050) accessible via Telegram (@zhuangba_openclaw_1st_bot). It uses `gpt-5.3-codex-high` model via a third-party API relay. Server is in mainland China and uses mihomo proxy for external access.

## Reading Order

| Priority | File | When to Read |
|----------|------|-------------|
| 1 | `*_setup-summary.md` | Always — core config and architecture |
| 2 | `*_server-env.md` | When touching server, network, or proxy |
| 3 | `*_evaluation.md` | When assessing or improving Jonathan |
| 4 | `*_token-analysis.md` | When optimizing cost or performance |
| 5 | `*_credentials.md` | Only when credentials are needed (SENSITIVE) |

## File Naming Convention
```
YYYY-MM-DD_topic-name.md
```
Each file has YAML frontmatter with `tags`, `depends_on`, and `status` for programmatic reference.

## Key Facts (TL;DR)
- Server: 192.168.0.18 (LAN) / 100.79.146.9 (Tailscale)
- Proxy: mihomo on port 7890, auto-start
- Gateway: port 18789, loopback only, SSH tunnel for access
- Jonathan scored 7/10 overall: good executor, weak on proactive memory
- First session used 326.6K tokens across 6 user messages
- Telegram Bot token is COMPROMISED and should be rotated
