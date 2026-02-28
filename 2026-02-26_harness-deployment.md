---
title: Harness 部署记录
date: 2026-02-26
last_updated: 2026-02-28
tags: [harness, deployment, mdie, monitoring]
depends_on: [2026-02-24_setup-summary.md]
status: current
---

# Harness 部署记录 — 2026-02-26

## 部署概要

Harness（自动化编码 agent）已部署到 Jonathan 服务器，Jonathan 现在可以编排 Harness 来开发完整项目。

| 项目 | 值 |
|------|-----|
| **安装路径** | ~/projects/harness-openai/ |
| **Python venv** | ~/projects/harness-openai/venv/ |
| **依赖** | aiohttp 3.13.3, playwright 1.58.0 |
| **启动脚本** | ~/projects/harness-openai/run_harness.sh |
| **操作手册** | ~/.openclaw/workspace/ 下三个文件：PLAYBOOK.md（入口）、APP_SPEC_GUIDE.md（写 spec）、MDIE.md（监控循环） |
| **入口** | autonomous_agent_demo.py |

## 环境变量 (.env)

| 变量 | 值 |
|------|-----|
| OPENAI_BASE_URL | https://right.codes/codex/v1 |
| OPENAI_MODEL | gpt-5.3-codex-high |
| BROWSER_HEADLESS | true |
| LOG_NATURAL_ONLY | true |

## 代码修改

1. **client.py L141**: `aiohttp.ClientSession(trust_env=True)` — 使 API 调用走 mihomo 代理
2. **initializer_prompt.md**: issue 数量从固定 100 改为按项目复杂度自适应（简单 5-15，中等 30-60，复杂 60-100）

## 渐进式披露设计

避免 Harness 知识污染每次对话的 system prompt：

| 文件 | 新增内容 | token 开销 |
|------|----------|-----------|
| AGENTS.md | 3 行：有 Harness 编排能力，需要时 `cat PLAYBOOK.md` | ~80 |
| TOOLS.md | 2 行：Harness 部署路径 + 指向 PLAYBOOK.md | ~40 |
| MEMORY.md | 1 行：Harness 已部署 | ~20 |
| **PLAYBOOK.md** | 入口路由（~45 行），指向 APP_SPEC_GUIDE.md 和 MDIE.md | 0 |
| **APP_SPEC_GUIDE.md** | 写 spec 指南（~100 行），按需 `cat` 读取 | 0 |
| **MDIE.md** | 监控循环（~130 行），HEARTBEAT 触发时读取 | 0 |

## MDIE 监控循环

HEARTBEAT.md 已配置，heartbeat 间隔 30 分钟。逻辑：
- 检测 `tmux list-sessions | grep harness-`
- 有 Harness 运行 → 执行监控、判断汇报/干预
- 无 Harness → HEARTBEAT_OK

MDIE 命令验证结果（全部通过）：

| 操作 | 状态 |
|------|------|
| Monitor: tmux 存活检查 | ✅ |
| Monitor: issues.json 进度解析 | ✅ |
| Monitor: git log | ✅ |
| Monitor: 日志读取 | ✅ (coding session 才生成 logs/) |
| Intervene L1: 软干预 comments.json | ✅ |
| Intervene L2: tmux 启停控制 | ✅ |
| Intervene L3: coding_prompt.md 追加 | ✅ |
| Intervene L4: git revert | ✅ |

## 端到端测试结果

测试项目 ~/projects/test-harness/（温度转换器）：
- 100 issues 创建成功（initializer session）
- converter.py 生成，代码质量良好
- git init + commit 成功
- 遇到 HTTP 500 时自动重试
- max-iterations=1 正确限制只跑初始化

## 待验证事项

- [x] 实弹测试：Telegram → Jonathan 读 PLAYBOOK → 执行 Harness 操作 ✅ D5 (2/28)
- [x] MDIE 精确监控：issues.json 解析 + git log + tmux 状态 ✅ D5（修复 PLAYBOOK 后）
- [ ] HEARTBEAT 驱动：Harness 运行中 Jonathan 被 HEARTBEAT 唤醒并自动监控
- [ ] 完整闭环：壮爸提需求 → spec → Harness → MDIE → coding 完成 → 汇报
- [ ] Coding session 质量：验证 Harness 编码阶段产出的代码是否可用

## PLAYBOOK 修复记录 (2/28)

**问题**：PLAYBOOK 原文"按需加载"+ "HEARTBEAT 触发时也读"→ Jonathan 理解为手动监控不需要 MDIE.md，只用 capture-pane 粗糙轮询
**修复**：改为"启动 Harness 后必须立即 cat MDIE.md"+ 警告不要只靠 capture-pane
**验证**：修复后 Jonathan 立即读取 MDIE.md 并使用精确监控命令，回复数据完全准确
