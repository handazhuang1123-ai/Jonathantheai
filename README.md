---
title: OpenClaw Jonathantheai - Memory Index
created: 2026-02-24
last_updated: 2026-03-03
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
| `docs/*_dissertation-harness-plan.md` | **论文流水线方案**：Harness 架构泛化为研究写作流水线（文献扫描→精读→起草→审查），含最佳实践和实施路线 | 涉及用 Harness 辅助博士论文研究写作时 |

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
- [可变] Jonathan 综合评分 7/10（D8 ↑↑），reasoning:true 修复效果显著。D8 核心交付：multi-agent 架构搭建。持续问题：message target 76 次错误、git 未提交
- [固定] 首次会话消耗 326.6K tokens / 6 条用户消息
- [可变] D4 错误：76x HTTP 502（模型端点不稳定）+ 9x message 参数回归 + 2x 对话卡死
- [可变] 服务器监控系统已部署（~/monitor/，cron 每天 08:00），推送已修复（D4，PATH 兜底）
- [可变] mail-assistant 已切换到 163 POP3 收件（IMAP 被风控拦截），fetch+分类+入库验证通过
- [可变] Proton Bridge keychain 已修复（D4），但邮件方案已改用 163，Bridge 暂不使用
- [固定] Telegram Bot token 泄露问题 — 不处理（壮爸决定）
- [可变] right.codes 余额耗尽（3/1 403），MiniMax fallback 实战验证通过（cooldown 机制自动跳过失败 provider），充值后自动恢复主模型
- [可变] MiniMax 用量：本地估算不可靠（reasoning tokens 不记录在 JSONL），改为余额手动汇报制（壮爸告知→Jonathan 记录→日报展示差值）
- [可变] OpenClaw v2026.2.26 升级（3/2）：External Secrets 已激活，两个 provider reasoning:true，x-tweet-fetcher skill 已注册
- [可变] Gateway 重启必须由外部执行（壮爸手动 systemctl restart），Jonathan 无法安全重启自己的 gateway
- [可变] HEARTBEAT v4 全面修复（3/2）：Telegram 发送修复 + per-project 告警抑制（JSON） + 显式项目注册 + META 排除 + 完成自动清除。Docker 端到端验证通过（docker-test 4/4 Done）
- [可变] 壮爸侧 `.claude/rules/` 模块化规则 + `CLAUDE.local.md` 自动加载已启用（2/27 验证通过）
- [可变] Harness 已部署到服务器（~/projects/harness-openai/），3/2 起 Docker 沙盒运行（文件系统隔离，MDIE 零修改）
- [可变] HEARTBEAT 已配置 MDIE 循环（每 30 分钟），但 D4 因 502 全部空转
- [可变] D5 已验证：Telegram 实弹测试 ✅、Harness 编排流程 ✅、MDIE 精确监控 ✅（修复 PLAYBOOK 后）、完整闭环 ✅（guess-number 11/12 Done）、coding 质量 ✅（9 commit，代码清晰）
- [可变] HEARTBEAT 自动驱动 MDIE 已验证（2/28，todo-cli 测试，评分 3/5：发现 crash ✅，卫生漏检 ❌，未干预 ❌，已二次修复 MDIE.md）
- [可变] 邮件三连已完成（3/1）：POP3 bug fix ✅ + Telegram 推送已开启 ✅ + full body storage 已加 ✅
- [可变] workspace 手册全面修复（2/28）：PLAYBOOK 加 spec 确认、MDIE 加 sleep 30s 上限+汇报触发+超时预警、HEARTBEAT 直接指向 MDIE
- [可变] 质量监控修复（2/28）：MDIE 加 [Q] Quality Check + coding_prompt 加卫生规则。手动触发 4/4 通过；HEARTBEAT 触发 3/5（卫生缓存+未干预）。**二次修复**：必须重新执行命令 + crash 后必须 L1 干预
- [可变] Oura Ring 健康数据接入（3/1）：OAuth2 认证通过，~/monitor/oura-sync.py cron 每天 08:10 同步睡眠/恢复/活动/血氧/压力到 ~/.oura/data/，Telegram 日报推送已验证，Jonathan workspace 已感知
- [可变] Gateway 自杀事件（3/1）：MiniMax 驱动的 Jonathan 建议执行 gateway stop 导致死循环。修复：归档有毒 session + workspace 第 9 条禁止建议停止自身 gateway
- [可变] D6 workspace 优化（3/1）：PLAYBOOK 加 Git 提交规范、HEARTBEAT 加 daily memory 检查。待验证：git 规范 + daily memory 生成
- [可变] minimax cost 配置已删除（3/2），成本用余额手动汇报制；custom-minimax 历史遗留已确认清理完毕
- [可变] 四 agent 体系（3/3）：main（Jonathan 🤖）+ gatekeeper（看门老大爷 🚬）+ nurse（性感小护士 🫦）+ teacher（大拿 🔧），各有独立 workspace + Telegram bot + 记忆体系
- [可变] 大拿 🔧（teacher agent，3/3 新增）：通用教学陪跑教练，边执行边科普。SOUL 定义人格，missions/current.md 定义具体任务（可替换），MEMORY 记录壮爸学习轨迹。近期任务：demo → 上线
- [可变] IDENTITY.md 是平台层元数据，不在 LLM 启动序列中。emoji 必须写入 SOUL.md 才能被 LLM 使用（3/3 已修复三个 agent）
- [可变] scheduler agent 已重命名为 gatekeeper（3/3）：openclaw.json + 目录 + cron + 脚本 + bin wrapper 全部更新，gateway 已重启验证
