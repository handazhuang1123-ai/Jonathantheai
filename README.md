---
title: OpenClaw Jonathantheai - Memory Index
created: 2026-02-24
last_updated: 2026-02-25
owner: zhuangba (壮爸)
purpose: Context transfer for new Claude sessions about Jonathan (OpenClaw agent)
---

# OpenClaw Jonathantheai - 记忆索引

本目录记录 **Jonathan**（OpenClaw AI agent）在壮爸 Linux 服务器上的运行、调试与优化过程。

## 快速背景

Jonathan 是一个 OpenClaw agent，通过 Telegram Bot 与用户交互，运行在中国大陆的 Linux 服务器上，依赖代理访问外部 API。具体版本、模型、硬件配置等详见 `*_setup-summary.md`。

## 阅读顺序

| 优先级 | 文件 | 何时加载 |
|--------|------|---------|
| 1 | `*_setup-summary.md` | 始终加载 — 核心配置与架构 |
| 2 | `*_server-env.md` | 涉及服务器、网络、代理、连接问题、服务状态时 |
| 3 | `*_evaluation.md` | 涉及评估、改进 Jonathan 的能力或行为时（`/jonathan-evaluate` 会自动加载作为历史基线） |
| 4 | `*_token-analysis.md` | 涉及 token 消耗、成本、性能优化时 |
| 5 | `*_credentials.md` | 仅在明确需要凭据时加载（敏感文件，已被 .gitignore 排除） |

> **注意**：以上 "何时加载" 是指导性描述而非精确匹配条件。判断时应结合用户意图的语义，不要拘泥于字面措辞。

## 文件命名规范

```
YYYY-MM-DD_topic-name.md
```

每个文件包含 YAML frontmatter：`tags`、`depends_on`、`status`、`last_updated`，用于渐进式加载和变更归属判断。

## 关键事实 (TL;DR)

<!-- 每条标注 [固定] 或 [可变]，方便 update-memory 时判断是否需要更新 -->
- [可变] 服务器: 192.168.0.18 (LAN) / 100.79.146.9 (Tailscale)
- [可变] 代理: mihomo 端口 7890，自启动
- [可变] Gateway: 端口 18789，仅 loopback，通过 SSH 隧道访问
- [可变] Jonathan 综合评分 7/10：能力面拓宽，工具鲁棒性是新短板
- [固定] 首次会话消耗 326.6K tokens / 6 条用户消息
- [可变] Day 2 会话消耗 138K tokens（cache 命中 91%），但 43 次错误（message 参数 + timeout）
- [可变] 服务器监控系统已部署（~/monitor/，cron 每天 08:00，GitHub: Jonathantheai/monitor-2026-02-25）
- [可变] Telegram Bot token 已泄露，需轮换
- [可变] 语音消息下载失败、message 工具参数兼容问题待修复
