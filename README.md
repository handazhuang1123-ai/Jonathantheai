---
title: OpenClaw Jonathantheai - Memory Index
created: 2026-02-24
last_updated: 2026-02-28
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
| 6 | `*_harness-deployment.md` | 涉及 Harness 部署状态、MDIE 循环、待验证事项时 |

> **注意**：以上 "何时加载" 是指导性描述而非精确匹配条件。判断时应结合用户意图的语义，不要拘泥于字面措辞。

## 参考文档 (docs/)

深度分析报告和构建计划，不是日常操作记忆。按需整篇加载，不纳入渐进式加载流程。

| 文件 | 说明 | 何时加载 |
|------|------|---------|
| `docs/*_agent-swarm-analysis.md` | Agent Swarm 架构深度分析（Elvis 模式、技术路线、分阶段方案） | 涉及多 agent 编排架构设计时 |
| `docs/*_harness-integration-insights.md` | Harness 集成洞察（Harness 如何替代 Claude Code CLI、架构变革） | 涉及 Harness 与 Jonathan 集成方案时（依赖上一条） |
| `docs/*_harness-build-plan.md` | **构建计划（主文档）**：部署步骤、MDIE 循环、app_spec 最佳实践、动态 Prompt | 需要在 Jonathan 服务器上构建 Harness 系统时（自包含，可独立阅读） |

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
- [可变] Jonathan 综合评分 7/10（D1-D3 持平），D4 待正式打分，D5 监控层测试通过
- [固定] 首次会话消耗 326.6K tokens / 6 条用户消息
- [可变] D4 错误：76x HTTP 502（模型端点不稳定）+ 9x message 参数回归 + 2x 对话卡死
- [可变] 服务器监控系统已部署（~/monitor/，cron 每天 08:00），推送已修复（D4，PATH 兜底）
- [可变] mail-assistant 已切换到 163 POP3 收件（IMAP 被风控拦截），fetch+分类+入库验证通过
- [可变] Proton Bridge keychain 已修复（D4），但邮件方案已改用 163，Bridge 暂不使用
- [可变] Telegram Bot token 已泄露，需轮换
- [可变] right.codes 模型端点不稳定（76x 502/天），heartbeat 空转，需监控或备用方案
- [可变] 壮爸侧 `.claude/rules/` 模块化规则 + `CLAUDE.local.md` 自动加载已启用（2/27 验证通过）
- [可变] Harness 已部署到服务器（~/projects/harness-openai/），端到端测试通过
- [可变] HEARTBEAT 已配置 MDIE 循环（每 30 分钟），但 D4 因 502 全部空转
- [可变] D5 已验证：Telegram 实弹测试 ✅、Harness 编排流程 ✅、MDIE 精确监控 ✅（修复 PLAYBOOK 后）、完整闭环 ✅（guess-number 11/12 Done）、coding 质量 ✅（9 commit，代码清晰）
- [可变] 待验证：HEARTBEAT 自动驱动 MDIE（Harness 运行中被 HEARTBEAT 唤醒并自动监控）
- [可变] 待操作：邮件 Telegram 摘要推送开启 + POP3 bug fix commit + 发测试邮件验证增量
- [可变] workspace 手册全面修复（2/28）：PLAYBOOK 加 spec 确认、MDIE 加 sleep 30s 上限+汇报触发+超时预警、HEARTBEAT 直接指向 MDIE
- [可变] 质量监控修复（2/28）：MDIE 加 [Q] Quality Check（代码运行+commit 审查+卫生+无限 session）、coding_prompt 加卫生规则、MEMORY 加质量经验
