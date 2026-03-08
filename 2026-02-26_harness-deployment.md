---
title: Harness 部署记录
date: 2026-02-26
last_updated: 2026-03-06
# fourth update 3/6: MDIE 监控职能分离到老大爷
# third update 3/2: Docker 沙盒部署
# second update 2/28: HEARTBEAT验证+二次修复
tags: [harness, deployment, mdie, monitoring]
depends_on: [2026-02-24_setup-summary.md]
status: current
---

# Harness 部署记录 — 2026-02-26

## 部署概要

Harness（自动化编码 agent）已部署到 Jonathan 服务器，**3/2 起在 Docker 容器中运行**（文件系统隔离）。

| 项目 | 值 |
|------|-----|
| **安装路径** | ~/projects/harness-openai/ |
| **运行方式** | Docker 容器（harness-sandbox 镜像，495MB） |
| **启动脚本** | ~/projects/harness-openai/run_harness.sh（Docker 封装版，原版 .bak 备份） |
| **操作手册** | PLAYBOOK.md + APP_SPEC_GUIDE.md（Jonathan workspace）、MDIE.md（Jonathan workspace，gatekeeper symlink 引用） |
| **入口** | autonomous_agent_demo.py |

### Docker 沙盒 (3/2)

容器挂载：`/harness` ← harness 代码（rw）、`/project` ← 目标项目（rw）、`/tmp` ← tmpfs 512MB。
容器不可见：~/.ssh、~/.openclaw、~/monitor、~/.oura、其他项目。
资源限制：2GB/2CPU（并发时 1.5GB/1.5CPU）、256 PIDs、`--security-opt no-new-privileges`。

关键文件：Dockerfile、docker-build.sh、.env.docker（从 .env 复制，.gitignore 已排除）。
日志路径修复：agent.py `project_dir.parent` → `Path("/harness/generations")`（避免写到容器内部丢失）。
Docker daemon 代理：`/etc/systemd/system/docker.service.d/proxy.conf`（大陆镜像加速源不可用，走 mihomo）。
构建命令：`docker build --network host --build-arg HTTP_PROXY=http://127.0.0.1:7890 --build-arg HTTPS_PROXY=http://127.0.0.1:7890 --tag harness-sandbox:latest .`
MDIE 零修改：tmux 包的是 docker run 前台进程，bind mount 双向可见，所有监控命令不变。
Jonathan workspace MEMORY.md 第 11 条已加。回滚：`cp run_harness.sh.bak run_harness.sh`。

## 环境变量 (.env.docker)

| 变量 | 值 |
|------|-----|
| OPENAI_BASE_URL | https://right.codes/codex/v1 |
| OPENAI_MODEL | gpt-5.3-codex-high |
| BROWSER_HEADLESS | true |
| LOG_NATURAL_ONLY | true |

> Docker 版使用 .env.docker（`--env-file`），代理地址 127.0.0.1:7890 通过 `--network host` 直通。

## 代码修改

1. **client.py L141**: `aiohttp.ClientSession(trust_env=True)` — 使 API 调用走 mihomo 代理
2. **initializer_prompt.md**: issue 数量从固定 100 改为按项目复杂度自适应（简单 5-15，中等 30-60，复杂 60-100）
3. **agent.py L565**: 日志路径修复 — Docker 环境下写到 `/harness/generations/`（非 Docker 保持原逻辑）

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

## MDIE 监控循环（3/6 移交老大爷）

**职能分离（3/6）**：MDIE 监控从 Jonathan 移交给老大爷（gatekeeper）。
- Jonathan：需求深挖 → 写 spec（含自检+3问速审）→ 启动 Harness → 注册到 `~/.openclaw/shared/harness-active-project.txt` → 任务完成
- 老大爷：heartbeat 检测活跃项目 → 读 MDIE.md（symlink）→ 执行 MDIE 全流程 → 汇报壮爸
- MDIE.md 源在 Jonathan workspace，老大爷通过 symlink 读取
- `harness-alert-count.json` 在老大爷 `workspace-gatekeeper/memory/`

heartbeat 间隔 30 分钟（老大爷执行）。逻辑：
- 检测 `shared/harness-active-project.txt` 和 `tmux list-sessions | grep harness-`
- 有 Harness 运行 → 执行 MDIE 监控（进度+质量+git）、判断汇报/干预
- 无 Harness 但有注册项目 → 检查完成率，汇报完成或异常停止
- 无活跃项目 → 跳过

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
- [x] HEARTBEAT 驱动：HEARTBEAT 唤醒 → 读 HEARTBEAT.md → 检测 harness tmux → 执行 MDIE 全流程（含 [Q]）✅（2/28 todo-cli 测试，评分 3/5：发现 crash 但卫生漏检+未干预）
- [x] 完整闭环：壮爸提需求 → spec → Harness → MDIE → coding 完成 → 汇报 ✅ D5（guess-number 项目，11/12 Done）
- [x] Coding session 质量：验证 Harness 编码阶段产出的代码是否可用 ✅ D5（main.py 功能完整、结构清晰、9 个 commit）
- [x] MDIE [Q] Quality Check：Jonathan 收到质量检查指令后自主执行 4/4 项检查 ✅（2/28 验证，发现 20 个 .png 散落、代码 EOF 容错不足、commit 无 regression、tmux 已停）

## 手册修复记录

**2/28 修复**（D5 实证驱动）：PLAYBOOK 加"必须立即 cat MDIE.md" + spec 确认环节。MDIE 加 sleep 上限 30s、每 3 Done 汇报、8 分钟超时。[Q] Quality Check 段落新增（代码运行/commit 审查/卫生/无限 session）。二次修复：失败必须干预 + 卫生禁复用缓存。

**3/1 D6 优化**：PLAYBOOK 加 Git 提交规范，HEARTBEAT 加 Daily Memory 检查。

**3/6 职能分离重构**：
- PLAYBOOK.md 重写：需求深挖清单（6项）+ spec 自检（6项）+ 3问速审 + 交接老大爷
- Jonathan HEARTBEAT.md：移除 Harness 监控（移交老大爷），保留 daily memory + git 卫生 + SHARED_RULES 审查
- 老大爷 HEARTBEAT.md 新建：Harness MDIE 监控 + 每日审计 + 告警抑制
- MDIE.md symlink 到老大爷，harness-active-project.txt 移到 shared/

**D7 HEARTBEAT 全面修复 + Docker 验证 (3/2)**：
- HEARTBEAT.md 重写（v4）：Telegram 发送修复（message 工具 target 参数 + JSON 示例）、显式项目注册（`memory/harness-active-project.txt` 替代目录扫描）、META issue 排除（priority=0 或 title 含 META）、完成后自动清除、per-project 告警抑制（`memory/harness-alert-count.json`，≥3 次跳过）
- 起因：D7 评估 + Docker 实测发现——机械重复告警、HEARTBEAT 回复不到 Telegram、旧项目误报、META 虚增完成率
- 告警中断事件：3/2 08:00~09:57（~2h），cron 在旧 gateway 下运行失败，重启后恢复
- Docker 端到端验证：docker-test 项目 4/4 Done（exit code 0），Jonathan MDIE 监控全流程正常
