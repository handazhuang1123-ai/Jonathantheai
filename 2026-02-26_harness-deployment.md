---
title: Harness 部署记录
date: 2026-02-26
last_updated: 2026-03-02
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
| **操作手册** | ~/.openclaw/workspace/：PLAYBOOK.md（入口）、APP_SPEC_GUIDE.md（写 spec）、MDIE.md（监控循环） |
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
- [x] HEARTBEAT 驱动：HEARTBEAT 唤醒 → 读 HEARTBEAT.md → 检测 harness tmux → 执行 MDIE 全流程（含 [Q]）✅（2/28 todo-cli 测试，评分 3/5：发现 crash 但卫生漏检+未干预）
- [x] 完整闭环：壮爸提需求 → spec → Harness → MDIE → coding 完成 → 汇报 ✅ D5（guess-number 项目，11/12 Done）
- [x] Coding session 质量：验证 Harness 编码阶段产出的代码是否可用 ✅ D5（main.py 功能完整、结构清晰、9 个 commit）
- [x] MDIE [Q] Quality Check：Jonathan 收到质量检查指令后自主执行 4/4 项检查 ✅（2/28 验证，发现 20 个 .png 散落、代码 EOF 容错不足、commit 无 regression、tmux 已停）

## 手册修复记录 (2/28)

**PLAYBOOK 修复**：
- "按需加载"→"启动 Harness 后必须立即 cat MDIE.md"+ 警告不要只靠 capture-pane
- 工作流加 spec 确认环节（壮爸批准后才启动）
- 加项目目录规范（~/projects/，壮爸指定名称）

**MDIE 修复**：
- sleep 上限 30s + 禁止递增
- 每 3 Done 必须汇报
- 8 分钟超时预警（主动汇报后停止循环）
- Done ≥ 90% + tmux 停止 → 必须立即汇报完成
- {project} 占位符说明 + issues.json 容错

**HEARTBEAT 修复**：
- 路径修正（直接引用 workspace 内文件）
- 有 Harness → 直接 cat MDIE.md（消除间接引用跳过风险）
- 汇报逻辑统一到 MDIE.md

**质量监控修复 (2/28)**：
- MDIE.md 新增 [Q] Quality Check 段落（Monitor 后必须执行）：代码可运行性、commit 变更审查、项目卫生、无限 session 检测
- coding_prompt.md 追加项目卫生规则（截图→screenshots/、完成后跑主入口、临时脚本用完即删）
- MEMORY.md 追加质量检查经验条目
- 根因同 D5 进度监控问题：MDIE 写了 Diagnose 模式但 Jonathan 从未执行（因为不是"必须"）

**质量监控二次修复 (2/28)**：
- [Q] 可运行性失败后必须立即执行 Level 1 干预（塞纸条通知编码 agent）
- [Q] 卫生检查必须重新执行命令，禁止复用上一轮结论
- 根因：HEARTBEAT 实弹测试中 Jonathan 发现 crash 只汇报不干预 + 卫生检查用旧缓存报"0"（实际 11 个）
- todo-cli 测试项目（~/projects/todo-cli/）已停止（Done=5/30），可清理
- initializer 对 todo-cli 创建 30 issues（偏多），自适应上限仍需调优

**D6 手册优化 (3/1)**：
- PLAYBOOK.md 新增"Git 提交规范"段落（每 commit 单一改动、message 与内容一致、先 diff --staged、禁止 git add .）
- HEARTBEAT.md 新增"Daily Memory 检查"段落（检测最新 daily 日期，繁忙日必须创建）
- 起因：D6 评估发现 git 规范退化（2/3 commit 有问题）+ 2/28-3/1 零 daily memory

**D7 HEARTBEAT 全面修复 + Docker 验证 (3/2)**：
- HEARTBEAT.md 重写（v4）：Telegram 发送修复（message 工具 target 参数 + JSON 示例）、显式项目注册（`memory/harness-active-project.txt` 替代目录扫描）、META issue 排除（priority=0 或 title 含 META）、完成后自动清除、per-project 告警抑制（`memory/harness-alert-count.json`，≥3 次跳过）
- 起因：D7 评估 + Docker 实测发现——机械重复告警、HEARTBEAT 回复不到 Telegram、旧项目误报、META 虚增完成率
- 告警中断事件：3/2 08:00~09:57（~2h），cron 在旧 gateway 下运行失败，重启后恢复
- Docker 端到端验证：docker-test 项目 4/4 Done（exit code 0），Jonathan MDIE 监控全流程正常
