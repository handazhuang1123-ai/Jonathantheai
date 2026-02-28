---
title: Harness 集成洞察 — 重新定义 Jonathan Agent Swarm
date: 2026-02-26
author: Claude Opus 4.6 (assisting zhuangba)
tags: [agent-swarm, harness, integration, architecture, insight]
depends_on: [2026-02-26_agent-swarm-analysis.md]
status: draft
last_updated: 2026-02-26
---

# Harness 集成洞察 — 重新定义 Jonathan Agent Swarm

> 壮爸已经跑通了一个 24/7 的 harness 流程，完成了 185 个 issue 的项目。这不是理论——这是实战数据。

---

## 核心洞察：Harness 就是更好的 Worker Engine

之前的 `agent-swarm-analysis.md` 设想用 **Claude Code CLI** (`claude -p`) 作为 worker agent。但壮爸手里已经有一个**更成熟、更可控、更省钱**的 worker：

**Harness（autonomous_agent_demo.py）不是 Claude Code 的替代品——它是 Claude Code 的上位替代。**

| 维度 | Claude Code CLI (旧方案) | Harness (新方案) |
|------|-------------------------|-----------------|
| 会话管理 | 无（headless 跑完就退出） | 自动多 session 循环，跑完所有 issue 才停 |
| 终止检测 | 无（靠 --max-turns 硬限） | **5 层检测**：END_SESSION + idle loop + tail injection + no-tool + max turns |
| Issue 追踪 | 无（需要外部 tasks.json） | **内建** local_issue_store，JSON 原子操作 |
| 质量门控 | 无 | `_has_written_code` 门控 + idle 检测，防止空跑 |
| 浏览器测试 | 不支持 | **18 个 Playwright 工具**，多 session 双浏览器 |
| 安全 | `--dangerously-skip-permissions` 或逐个确认 | **命令白名单** + 敏感命令验证 |
| 内存风险 | 已知泄漏（1 GB/min 回归） | Python 进程，无 Node.js 内存泄漏问题 |
| API | 需要 Anthropic API key | **用 right.codes**（同 Jonathan，$0 额外成本） |
| 跨 session 记忆 | 无 | issue comments + git log + META issue |
| 代码量 | Anthropic 闭源 | **3,768 行 Python，完全可控** |
| 实战验证 | 单次任务 | **185 issue，100% 完成率** |

**结论：Harness 已经解决了 agent-swarm-analysis.md 中 Phase 1-2 的大部分难题。**

---

## 这如何改变架构

### 旧架构（agent-swarm-analysis.md）

```
壮爸 → Jonathan → tmux → claude -p "prompt" → 手动监控 → PR
                         ↑
                    需要从零构建:
                    - 任务注册表
                    - 看门狗
                    - 终止检测
                    - 重试逻辑
```

### 新架构（Harness 集成）

```
壮爸 → Jonathan → tmux → harness (python autonomous_agent_demo.py)
       (编排者)           (worker engine — 已经内建一切)
         │                    │
         │                    ├── 自动拆解 app_spec → 100+ issues
         │                    ├── 多 session 自动循环
         │                    ├── 5 层终止检测
         │                    ├── 浏览器验证
         │                    ├── idle loop 防空跑
         │                    └── git commit 每个 issue
         │
         ├── 监控 .issue_store/issues.json（进度）
         ├── 监控 logs/（日志）
         └── Telegram 通知壮爸
```

**Jonathan 的角色从"管理每一步"变成"写好 spec 然后看着它自己跑"。**

这正是 Elvis 描述的 Zoe 模式：
> "Zoe spawns the agents, writes their prompts, picks the right model, monitors progress, and pings me on Telegram when PRs are ready."

区别是：Elvis 用 Codex/Claude Code 作为 worker；**我们用 Harness**——一个已经证明能跑完 185 issue 的自主编码引擎。

---

## 三个创意方案

### 方案 A：Jonathan 远程编排 WSL 上的 Harness

```
Jonathan (服务器 192.168.0.18)
    │
    │ SSH / Tailscale
    ▼
壮爸的 WSL (Ubuntu 24.04)
    │
    └── tmux: harness-{project}
        python autonomous_agent_demo.py --project-dir ./generations/{project}
```

**优点**：
- WSL 有更多 RAM（壮爸的 Windows 机器）
- Chromium/Playwright 跑在 WSL 上，不占 Jonathan 服务器资源
- Jonathan 服务器只做编排（极低资源消耗）
- Harness 已在 WSL 上验证过

**缺点**：
- 需要 Jonathan 能 SSH 到 WSL（反向 SSH 或 Tailscale）
- 两台机器的依赖

**适合**：重度项目（需要浏览器测试、大量编码）

### 方案 B：Harness 部署到 Jonathan 服务器本地

```
Jonathan (服务器)
    │
    └── tmux: harness-{project}
        python autonomous_agent_demo.py --project-dir ~/projects/{project}
```

**资源评估**：
- Python 进程 + aiohttp：~50-100 MB RAM
- Playwright + Chromium（如果需要浏览器测试）：~200-400 MB RAM
- 无浏览器模式（纯编码项目）：~50-100 MB RAM
- **对比 Claude Code CLI 的 256 MB - 2 GB → 更省**

**优点**：
- 一切在同一台机器，简单
- Jonathan 直接用 shell 工具操作，无需 SSH
- 共享 right.codes API 配置

**缺点**：
- 浏览器测试会占 Jonathan 服务器资源
- 需要安装 Python 依赖 + Playwright + Chromium

**适合**：不需要浏览器测试的项目（脚本、CLI 工具、后端服务）

### 方案 C：混合模式（推荐）

```
Jonathan (服务器) — 编排 + 轻量任务
    │
    ├── 本地 harness (无浏览器)：脚本、CLI、后端
    │   └── BROWSER_HEADLESS=true + 不安装 Chromium
    │       纯编码模式，~50 MB RAM
    │
    └── SSH → WSL harness (有浏览器)：Web 应用
        └── 完整 Playwright + Chromium
            浏览器验证 + 截图

Jonathan 根据 app_spec 内容判断：
- 有 UI？→ 发到 WSL
- 纯后端/脚本？→ 本地跑
```

---

## 关键集成点：Jonathan 如何控制 Harness

### 1. Jonathan 生成 app_spec.txt

这是整个系统的起点。壮爸通过 Telegram 描述需求 → Jonathan 把模糊需求转化为结构化的 `app_spec.txt`。

```
壮爸: "我要一个能自动整理微信账单的工具"
        ↓
Jonathan (用自己的 LLM 能力):
  → 分析需求
  → 查阅 memory/ 了解壮爸的技术偏好
  → 生成 app_spec.txt
  → 确认后启动 harness
```

**这就是 Elvis 说的"上下文分离"**：
- Jonathan 持有业务上下文（壮爸是谁、用什么技术栈、过去做过什么）
- Harness/Worker 只看 app_spec.txt 和代码

### 2. Jonathan 启动 Harness

```bash
# Jonathan 通过 exec 工具执行
tmux new-session -d -s harness-billing \
  -c "/home/zhuangba/projects/harness-openai" \
  "python autonomous_agent_demo.py --project-dir ~/projects/billing-tool"
```

### 3. Jonathan 监控进度

```bash
# 读取 issue 进度（零 token，纯 JSON 解析）
cat ~/projects/billing-tool/.issue_store/issues.json | python3 -c "
import json, sys
issues = json.load(sys.stdin)
total = len(issues)
done = sum(1 for i in issues.values() if i['state']['name']=='Done')
in_progress = sum(1 for i in issues.values() if i['state']['name']=='In Progress')
print(f'进度: {done}/{total} ({done/total*100:.0f}%)')
if in_progress:
    for i in issues.values():
        if i['state']['name']=='In Progress':
            print(f'  正在进行: {i[\"title\"][:50]}')
"

# 检查 tmux 是否还在运行
tmux has-session -t harness-billing 2>/dev/null && echo "运行中" || echo "已完成"

# 读取最近日志
tail -20 ~/projects/billing-tool/logs/all_sessions.txt
```

### 4. Jonathan 通过 Telegram 汇报

```
Jonathan → 壮爸 (Telegram):
"📊 billing-tool 进度更新：
  完成: 45/120 (38%)
  正在进行: [RUN-46] 导入微信 CSV 解析
  预计还需 ~3 个 session

  最近完成:
  ✅ [RUN-43] 基础项目结构
  ✅ [RUN-44] 账单数据模型
  ✅ [RUN-45] CSV 文件上传 UI"
```

### 5. Jonathan 干预 Harness（当出问题时）

```bash
# 如果 harness 卡住了，发送中断
tmux send-keys -t harness-billing C-c

# 修改 app_spec 或 issue 后重新启动
# （harness 会从 .issue_store 恢复，不会从头开始）
tmux send-keys -t harness-billing \
  "python autonomous_agent_demo.py --project-dir ~/projects/billing-tool" Enter
```

---

## Harness 的已知局限（对集成的影响）

| 局限 | 影响 | Jonathan 的解决方案 |
|------|------|---------------------|
| 单线程（一次一个项目） | 不能同时跑多个项目 | Jonathan 顺序调度多个 harness 实例 |
| 无 PR 创建 | 代码只在本地 commit | Jonathan 在项目完成后执行 `gh pr create` |
| 无成本预算 | token 消耗无上限 | Jonathan 监控 session 数量，设软限制 |
| 无 CI 集成 | 只有浏览器测试 | Jonathan 在 PR 前运行 `npm test` / `npm run lint` |
| app_spec 质量决定一切 | 差的 spec → 差的结果 | **Jonathan 的核心价值就在于写好 spec** |
| 固定的 prompt 模板 | 不够灵活 | Jonathan 可以动态修改 coding_prompt.md |

---

## 与 agent-swarm-analysis.md 的关系

之前的分析**仍然有效**，但优先级和实施路径变了：

| agent-swarm-analysis 中的内容 | 新状态 |
|-------------------------------|--------|
| Phase 0: 基础设施加固（swap/zram/pnpm） | **仍然需要**（如果在服务器上跑 harness） |
| Phase 0: 安全加固（Bot token/UFW） | **仍然需要，优先级不变** |
| Phase 1: 单 Worker 验证 | **大幅简化** — harness 替代 claude -p |
| Phase 1: 编写看门狗 | **简化** — Python 进程比 Node.js 稳定得多 |
| Phase 1: 编写 tasks.json | **不需要** — harness 内建 issue_store |
| Phase 2: check-agents.sh | **简化** — 只需检查 tmux + 读 issue_store |
| Phase 2: Ralph Loop V2 重试 | **部分内建** — harness 有 idle 检测和自动重试 |
| Phase 3: 任务拆分 | **自动化** — harness 的 initializer agent 自动拆 100+ issue |
| Phase 3: Worker prompt 模板 | **已有** — coding_prompt.md (565 行) + initializer_prompt.md (197 行) |
| memory-guard.sh | **仍然建议有**，但风险大幅降低 |

**总结：之前要从零构建的 60% 工作量，harness 已经做完了。**

---

## 新的实施路径（精简版）

### Step 1: 在 Jonathan 服务器上部署 Harness（~1 小时）

```bash
# 在服务器上
cd ~/projects
git clone <harness-repo> harness-openai
cd harness-openai
pip install -r requirements.txt
# 如果需要浏览器测试：playwright install chromium

# 配置 API（已有 right.codes）
export OPENAI_API_KEY="sk-e1c6cb1a5c0144fd98581b0da9391cc6"
export OPENAI_BASE_URL="https://right.codes/codex/v1"
export OPENAI_MODEL="gpt-5.3-codex-high"
```

### Step 2: 手动验证 Harness 能在服务器上跑（~30 分钟）

```bash
# 用一个极简 app_spec 测试
echo "Build a simple Python CLI that converts Celsius to Fahrenheit" > prompts/app_spec.txt
python autonomous_agent_demo.py --project-dir ./generations/test-cli --max-iterations 2
```

### Step 3: 教 Jonathan 如何编排 Harness（~2 小时）

更新 Jonathan 的 workspace 文件：
- `TOOLS.md`：添加 harness 的启动/监控/干预命令
- `AGENTS.md`：添加编排者角色描述
- `MEMORY.md`：添加 harness 使用经验

### Step 4: 端到端测试（~1 小时）

```
壮爸 → Jonathan (Telegram): "用 harness 帮我做一个简单的 todo list web app"
Jonathan 应该：
1. 生成 app_spec.txt
2. 启动 harness
3. 定期通过 Telegram 汇报进度
4. 完成后执行 git push + PR
5. 通知壮爸
```

---

## 更深层的创意启发

### 1. "Jonathan + Harness" 是 Zoe 模式的最优中国大陆实现

Elvis 的 Zoe 系统需要：Anthropic API ($100/月) + OpenAI Codex ($90/月) + 128GB Mac Studio ($3,500)。

壮爸的系统：right.codes ($0-低成本) + 现有服务器 ($0) + 现有 WSL ($0)。**成本可以是 Elvis 的 1/10 甚至更低**，因为 harness 用的是同一个已有的 API。

### 2. app_spec.txt 是新的"Obsidian vault"

Elvis 的 Zoe 从 Obsidian vault 获取业务上下文。Jonathan 的等价物是：

```
壮爸的需求 (Telegram)
    → Jonathan 的理解 (memory/ + SOUL.md 中的偏好)
        → app_spec.txt (结构化需求文档)
            → Harness 自动拆解为 100+ issue
                → 自动编码
```

**app_spec.txt 的质量决定项目成败**。Jonathan 的核心竞争力不是写代码——而是**理解壮爸、写出好的 spec**。这完美匹配 Jonathan 的优势（指令遵循 8/10、概念解释 8/10）和回避了它的短板（工具调用鲁棒性 7/10）。

### 3. Harness 的 5 层终止检测治好了 Jonathan 的"暴力重试病"

Jonathan Day 2 最严重的问题是 40 次用相同参数重试 message 工具。Harness 的 idle loop 检测（5 轮空闲 → 强制结束，10 轮 → 硬退出）正好解决这个问题——**不是在 Jonathan 层面修，而是让 worker 自己有熔断器**。

### 4. Issue Store 是天然的"通信协议"

Jonathan 和 Harness 之间不需要复杂的 IPC 机制。`.issue_store/issues.json` 就是两者的共享状态：

- Harness 写入（创建 issue、更新状态、添加 comment）
- Jonathan 只读（监控进度、生成报告）
- **无竞态**（Harness 是唯一写入者，Jonathan 是只读观察者）

### 5. 渐进式扩展路径清晰

```
Level 1: Jonathan + 1 个 Harness（当前目标）
         壮爸说"做 X" → Jonathan 写 spec → 启动 harness → 通知完成

Level 2: Jonathan + 多个 Harness（顺序）
         壮爸说"做 X 和 Y" → Jonathan 先跑 X 的 harness，完成后跑 Y

Level 3: Jonathan + 多个 Harness（并行，跨机器）
         X 跑在服务器（无浏览器），Y 跑在 WSL（有浏览器）

Level 4: Jonathan 主动发现工作
         扫描 GitHub issues → 生成 app_spec → 启动 harness
         （= Elvis 的 "Zoe 早上扫描 Sentry" 模式）
```

### 6. Harness 可以定制和进化

3,768 行 Python，完全开源，壮爸完全掌控。可以做的改进：

| 改进 | 实施难度 | 价值 |
|------|----------|------|
| 添加 `gh pr create` 到编码流程 | 低（security.py 加白名单 + coding_prompt 加步骤） | 高 |
| 添加 `npm test` / `npm run lint` 步骤 | 低 | 中 |
| 让 Jonathan 动态修改 coding_prompt.md | 中 | 高（Ralph Loop V2） |
| 添加进度 webhook（harness → Jonathan） | 中 | 高（实时通知） |
| 支持多 worktree 并行 | 高（需要重构 agent.py） | 中（服务器 RAM 限制并行） |
| 添加 token 预算控制 | 低（client.py 加计数器 + 阈值） | 中 |

### 7. 壮爸已经有了一个"一人开发团队"

Elvis 说"My git history looks like I just hired a dev team."

壮爸的 MTG Playtable 项目：**185 个 issue，100% 完成**。这已经是一个一人开发团队的产出了。

区别只是：
- Elvis 的系统通过 Telegram 通知（Jonathan 也能做）
- Elvis 有 PR + code review 流程（harness 可以加）
- Elvis 能并行 4-5 个 agent（壮爸暂时顺序）

**Harness 是已经验证的 worker。Jonathan 是缺失的编排层。两者结合就是完整的 Zoe 系统。**

---

## 对 agent-swarm-analysis.md 的修正建议

1. **降低 Phase 1 复杂度** — 不需要从零写 tasks.json、看门狗、终止检测。Harness 全有。
2. **增加 "Harness 部署" 步骤** — 在 Phase 0.5 把 harness 部署到服务器。
3. **重新定义 Jonathan 的角色** — 从"管理 claude -p worker"变成"写 spec + 启动 harness + 监控 + 通知"。
4. **降低安全风险** — Harness 有命令白名单，比 `--dangerously-skip-permissions` 安全得多。
5. **降低成本预估** — 不需要额外 Anthropic API key，right.codes 全覆盖。

---

## 壮爸的新决策点

之前 6 个决策点中，部分已被 Harness 简化：

| 旧决策 | 新状态 |
|--------|--------|
| D2: Worker API? | **已解决** — Harness 用 right.codes，与 Jonathan 同源 |
| D4: 安全级别? | **已解决** — Harness 有内建白名单，比 Claude Code 安全 |
| D5: 安全先修? | **仍然需要决定**（Bot token + UFW） |
| D6: 月度预算? | **大幅降低** — 不需要额外 API 费用 |

**新决策点**：

### N1: Harness 部署在哪？

| 选项 | 说明 |
|------|------|
| A) Jonathan 服务器本地 | 简单，但共享 7.6GB RAM |
| B) 继续在 WSL 跑，Jonathan SSH 编排 | 资源分离，但需要网络配置 |
| C) 混合（推荐） | 无浏览器任务在服务器，Web 项目在 WSL |

### N2: 第一个 Jonathan + Harness 联动项目是什么？

可以用 MTG Playtable 的经验，但需要一个**新的 app_spec**。

---

*本文档是对 `2026-02-26_agent-swarm-analysis.md` 的补充和修正，基于壮爸已验证的 Harness 系统。两份文档一起构成完整的 Jonathan Agent Swarm 实施方案。*
