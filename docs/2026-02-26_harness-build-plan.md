---
title: Jonathan + Harness 构建计划 — 完整实施指南
date: 2026-02-26
author: Claude Opus 4.6 (assisting zhuangba)
tags: [harness, build-plan, implementation, jonathan, agent-swarm]
depends_on: [2026-02-26_agent-swarm-analysis.md, 2026-02-26_harness-integration-insights.md, 2026-02-24_setup-summary.md, 2026-02-24_server-env.md]
status: current
last_updated: 2026-02-26
---

# Jonathan + Harness 构建计划 — 完整实施指南

> **本文档的唯一目的**：给一个全新的 context window 看完后，就能直接在 Jonathan 服务器上动手构建整个系统。
>
> **前置文档**（提供背景和决策过程，但本文档是自包含的）：
> - `2026-02-26_agent-swarm-analysis.md` — 原始 Agent Swarm 架构分析
> - `2026-02-26_harness-integration-insights.md` — Harness 如何改变架构

---

## 一、壮爸已做出的关键决策

这些决策不可更改，构建计划必须完全遵守：

| # | 决策 | 说明 |
|---|------|------|
| D1 | **只用方案 B：Harness 部署在 Jonathan 服务器本地** | 给 Jonathan 独立机器的意义就是不用壮爸的电脑。不要 SSH 到 WSL，不要混合模式。 |
| D2 | **单线程顺序执行** | Token 有每日上限，一天只能跑几个 session，暂不需要并行。 |
| D3 | **Jonathan 必须能自主完成监控-调试-实施-评估循环** | 185 个 issue 不是顺序跑完的，需要多次调试。这个调试过程以后由 Jonathan 执行。 |
| D4 | **app_spec 质量是核心竞争力** | 需要最佳实践框架，Jonathan 的核心价值是理解壮爸的需求并写出好的 spec。 |
| D5 | **不执行任何代码操作** | 本文档只是计划，不改变服务器上的任何内容。 |

---

## 二、系统全景图

```
壮爸 (Telegram)
    │
    │ "帮我做一个 XXX 应用"
    ▼
Jonathan (OpenClaw agent, 192.168.0.18)
    │
    ├── [1] 理解需求 → 生成 app_spec.txt
    │       │
    │       ├── 参考 memory/ 中壮爸的技术偏好
    │       ├── 参考 app_spec 最佳实践模板
    │       └── 通过 Telegram 与壮爸确认
    │
    ├── [2] 准备项目环境
    │       │
    │       ├── mkdir ~/projects/{project-name}
    │       ├── 写入 app_spec.txt
    │       └── (可选) 定制 coding_prompt.md
    │
    ├── [3] 启动 Harness
    │       │
    │       └── tmux new -d -s harness-{project}
    │           "cd ~/harness-openai && python autonomous_agent_demo.py
    │            --project-dir ~/projects/{project-name}"
    │
    ├── [4] 监控-调试-评估循环 ← 核心创新
    │       │
    │       ├── 定期检查 .issue_store/issues.json（进度）
    │       ├── 检查 logs/all_sessions.txt（质量）
    │       ├── 检查 tmux 是否还在运行（存活）
    │       ├── 发现问题 → 调试干预（见第五章）
    │       └── 通过 Telegram 向壮爸汇报
    │
    ├── [5] 项目收尾
    │       │
    │       ├── 运行 npm test / npm run lint（如适用）
    │       ├── git push + gh pr create
    │       └── 通知壮爸 review
    │
    └── [6] (高级) 多 worktree 对比
            │
            ├── 同一 app_spec，不同 model/prompt
            ├── worktree-A: gpt-5.3-codex-high
            ├── worktree-B: gpt-5.3-codex-low
            └── Jonathan 对比两者质量，选择更好的
```

---

## 三、Harness 部署步骤（Jonathan 服务器本地）

### 3.1 前提条件检查

```bash
# 检查 Python 版本（需要 3.10+）
python3 --version

# 检查可用 RAM（Harness 需要 ~50-100 MB，无浏览器模式）
free -h

# 检查磁盘空间（每个项目 ~100-500 MB）
df -h /home

# 检查 tmux（已安装 3.4）
tmux -V

# 检查 git（已安装 2.43.0）
git --version
```

### 3.2 安装 Harness

```bash
cd ~/projects

# 方案 A：从壮爸的 WSL 复制（推荐，保留已有配置）
# 壮爸在 WSL 上执行：
scp -r ~/projects/Claude\ Cole/Linear-Coding-Agent-Harness-openai/ \
  zhuangba@192.168.0.18:~/projects/harness-openai/

# 方案 B：如果有 git 仓库
git clone <harness-repo-url> harness-openai

cd harness-openai

# 安装 Python 依赖
pip install -r requirements.txt
# requirements.txt 内容：aiohttp>=3.9.0, playwright>=1.40.0

# 浏览器测试（可选，会占 200-400 MB RAM）
# 如果项目不需要浏览器测试，跳过这步以节省资源
# playwright install chromium
```

### 3.3 配置环境变量

```bash
# 在 harness-openai/.env 或 shell profile 中设置
export OPENAI_API_KEY="<壮爸的 right.codes key>"
export OPENAI_BASE_URL="https://right.codes/codex/v1"
export OPENAI_MODEL="gpt-5.3-codex-high"

# 可选：精简日志
export LOG_NATURAL_ONLY=true

# 可选：无浏览器模式（节省 RAM）
export BROWSER_HEADLESS=true
```

### 3.4 验证安装

```bash
# 创建一个极简测试项目
mkdir -p ~/projects/test-harness
echo "Build a simple Python CLI that converts Celsius to Fahrenheit.
Single file, no dependencies." > prompts/app_spec.txt

# 限制只跑 1 个 session（验证能启动）
python autonomous_agent_demo.py \
  --project-dir ~/projects/test-harness \
  --max-iterations 1

# 检查结果
ls ~/projects/test-harness/
cat ~/projects/test-harness/.issue_store/issues.json | python3 -c "
import json, sys
issues = json.load(sys.stdin)
print(f'创建了 {len(issues)} 个 issue')
for k, v in issues.items():
    print(f'  {k}: {v[\"title\"][:60]}')
"
```

### 3.5 资源预估

| 场景 | RAM 占用 | 说明 |
|------|----------|------|
| Harness (无浏览器) | ~50-100 MB | Python + aiohttp |
| Harness (有浏览器) | ~250-500 MB | + Playwright + Chromium |
| Jonathan (OpenClaw Gateway) | ~400 MB | 已有 |
| 系统 + VNC + mihomo | ~500 MB | 已有 |
| **总计 (无浏览器)** | **~1.0-1.1 GB** | 6.5 GB 剩余 |
| **总计 (有浏览器)** | **~1.2-1.5 GB** | 6.1 GB 剩余 |

**结论**：7.6 GB RAM 完全够用，甚至有浏览器也没问题。

---

## 四、app_spec.txt 最佳实践

> "app_spec 的质量决定项目成败" — 壮爸

### 4.1 研究总结

基于 Addy Osmani 的 "How to write a good spec for AI agents"、GitHub Spec Kit 的 SDD（Spec-Driven Development）流程、以及壮爸已成功的 MTG Playtable app_spec 分析：

**核心原则：规格越明确，Agent 的发挥空间越小，结果越可预测。**

### 4.2 app_spec 六大必备区域（Addy Osmani 框架）

| 区域 | 必须包含 | 壮爸 MTG spec 示例 |
|------|----------|-------------------|
| **1. 命令与脚本** | 构建、运行、测试的精确命令 | `npm run dev`, `npm run build`, `init.sh` 脚本 |
| **2. 测试策略** | 使用什么框架、怎么跑、覆盖率目标 | 浏览器测试 + Playwright 验证 |
| **3. 项目结构** | 目录树、文件命名约定 | 完整 `src/` 目录树 + 每个文件的职责说明 |
| **4. 代码风格** | 语言版本、lint 规则、命名约定 | TypeScript strict, React 19, Tailwind 4 |
| **5. Git 工作流** | 分支策略、提交消息格式 | master 分支 + feature/db-migration |
| **6. 边界与约束** | 不要做什么、技术限制 | "不要用 ORM"、"不要创建测试文件" |

### 4.3 app_spec 模板（Jonathan 生成 spec 时使用）

```markdown
# {项目名称} — 应用规格

## 项目概述
一句话描述这个应用是什么。

## 技术栈（精确到版本号）
- 语言: TypeScript 5.x / Python 3.12
- 框架: Next.js 15 / FastAPI 0.115
- 包管理: pnpm / pip
- 数据库: SQLite / PostgreSQL
- 样式: Tailwind CSS 4 / plain CSS

## 目录结构
```
project/
├── src/
│   ├── components/    # React 组件
│   ├── lib/           # 工具函数
│   ├── api/           # API 路由
│   └── types/         # TypeScript 类型
├── public/
├── package.json
└── tsconfig.json
```

## 核心数据模型
用 TypeScript interface 或 Python dataclass 精确定义：
```typescript
interface User {
  id: string;
  name: string;
  email: string;
  createdAt: Date;
}
```

## 功能列表（按优先级）

### P1 — 核心功能（必须有）
1. [功能名]：[详细描述，包括输入/输出/边界情况]
2. ...

### P2 — 重要功能
1. ...

### P3 — 锦上添花
1. ...

## UI/UX 规格
- 布局示意图（ASCII art 或文字描述）
- 颜色方案
- 响应式断点

## API 设计（如有后端）
```
GET    /api/users          → 获取用户列表
POST   /api/users          → 创建用户 { name, email }
PUT    /api/users/:id      → 更新用户
DELETE /api/users/:id      → 删除用户
```

## 命令
```bash
# 安装
pnpm install

# 开发
pnpm dev

# 构建
pnpm build

# 测试
pnpm test
```

## 约束与边界
- 不要使用 ORM，直接写 SQL
- 不要创建独立的测试文件
- 所有状态管理用 Zustand，不用 Redux
- 不要添加用户认证，这是内部工具
```

### 4.4 app_spec 质量检查清单

Jonathan 在提交 spec 给 Harness 之前，应自我检查：

- [ ] **技术栈有版本号**：不是 "React"，而是 "React 19.1"
- [ ] **目录结构完整**：不是 "src/ 下放组件"，而是完整的树形结构
- [ ] **数据模型有类型定义**：不是 "用户有名字"，而是 `interface User { name: string }`
- [ ] **API 端点有 HTTP 方法 + 路径 + 请求体/响应体**
- [ ] **有明确的"不要做什么"**：避免 Agent 过度工程化
- [ ] **命令可直接复制粘贴执行**：不是 "用 npm 跑"，而是 `npm run dev`
- [ ] **功能有优先级标注**：P1/P2/P3/P4，Harness 的 initializer 会据此分配 issue 优先级
- [ ] **UI 有布局描述**：ASCII art 或详细文字，不是 "做一个好看的界面"

### 4.5 从壮爸需求到 app_spec 的转化流程

```
壮爸 (Telegram): "我要一个能自动整理微信账单的工具"
        │
        ▼
Jonathan 思考：
  1. 壮爸的技术偏好是什么？→ 查 MEMORY.md
  2. 这个工具需要 UI 还是 CLI？→ 问壮爸
  3. 微信账单格式是什么？→ 搜索 + 问壮爸
  4. 类似工具有哪些？→ 搜索竞品
        │
        ▼
Jonathan 生成 app_spec 初稿
        │
        ▼
Jonathan 通过 Telegram 发给壮爸确认：
  "壮爸，这是我理解的需求：
   - Python CLI 工具
   - 读取微信账单 CSV
   - 分类统计
   - 输出 Excel 报表

   技术栈我选了 Python + pandas + openpyxl。

   完整 spec 我放在 [链接]，你看看有没有要改的？"
        │
        ▼
壮爸确认（或修改）→ Jonathan 定稿 → 启动 Harness
```

**关键**：Jonathan 不应该假设壮爸想要什么，而是**问清楚再写 spec**。模糊的需求 → 模糊的 spec → 差的代码。

---

## 五、监控-调试-实施-评估循环（MDIE Loop）

> 这是本构建计划的核心创新。壮爸说："185 个 issue 并不是顺序跑完，而是经过了多次调试。这个调试的过程以后由 Jonathan 实现。"

### 5.1 概念

```
Jonathan 的 MDIE Loop（每 N 分钟或每个 session 结束后触发）

    ┌─── [M] Monitor 监控 ───┐
    │                          │
    │  读取 .issue_store/      │
    │  读取 logs/              │
    │  检查 tmux 存活          │
    │                          │
    └────────┬─────────────────┘
             │
             ▼
    ┌─── [D] Diagnose 诊断 ───┐
    │                          │
    │  Session 是否成功？       │
    │  有没有卡住？             │
    │  代码质量如何？           │
    │  Issue 是否真正完成？     │
    │                          │
    └────────┬─────────────────┘
             │
        ┌────┴────┐
        │ 正常？   │
        └────┬────┘
         Yes │  No
             │   │
             │   ▼
             │  ┌─── [I] Intervene 干预 ───┐
             │  │                            │
             │  │  修改 coding_prompt.md     │
             │  │  修改 issue 描述           │
             │  │  重启 harness              │
             │  │  回退 git commit           │
             │  │                            │
             │  └────────┬──────────────────┘
             │           │
             ▼           ▼
    ┌─── [E] Evaluate 评估 ───┐
    │                          │
    │  Session 产出评分         │
    │  记录经验到 memory        │
    │  向壮爸汇报               │
    │  决定下一步               │
    │                          │
    └──────────────────────────┘
```

### 5.2 Monitor（监控）— Jonathan 执行的具体命令

```bash
# === 1. 进度监控 ===
# 读取 issue 进度（零 token 消耗，纯 JSON 解析）
cat ~/projects/{project}/.issue_store/issues.json | python3 -c "
import json, sys
issues = json.load(sys.stdin)
total = len(issues)
done = sum(1 for i in issues.values() if i.get('state',{}).get('name')=='Done')
in_progress = sum(1 for i in issues.values() if i.get('state',{}).get('name')=='In Progress')
todo = sum(1 for i in issues.values() if i.get('state',{}).get('name')=='Todo')
cancelled = sum(1 for i in issues.values() if i.get('state',{}).get('name')=='Cancelled')
print(f'进度: {done}/{total} ({done/total*100:.0f}%)')
print(f'  Done: {done}, In Progress: {in_progress}, Todo: {todo}, Cancelled: {cancelled}')
if in_progress:
    for i in issues.values():
        if i.get('state',{}).get('name')=='In Progress':
            print(f'  → 正在进行: [{i[\"identifier\"]}] {i[\"title\"][:60]}')
"

# === 2. 存活检查 ===
tmux has-session -t harness-{project} 2>/dev/null && echo "运行中" || echo "已停止"

# === 3. 最近日志 ===
tail -30 ~/projects/{project}/logs/all_sessions.txt

# === 4. Git 历史（最近 session 的产出）===
cd ~/projects/{project} && git log --oneline -10

# === 5. 最近 session 的 token 消耗 ===
grep -i "token" ~/projects/{project}/logs/all_sessions.txt | tail -5
```

### 5.3 Diagnose（诊断）— 问题模式识别

| 模式 | 检测方法 | 严重程度 | 典型原因 |
|------|----------|----------|----------|
| **Session 空跑** | issue 状态未变化 + git log 无新 commit | 高 | spec 不清晰、issue 描述不够具体 |
| **Session 卡死** | tmux 运行但 log 长时间无输出 | 高 | API 限流、网络问题、无限循环 |
| **Issue 被标 Done 但功能不工作** | 浏览器测试失败 / 代码不完整 | 中 | Agent 过早标完、测试不充分 |
| **同一 Issue 反复 In Progress** | 多个 session 认领同一 issue 但都没完成 | 中 | Issue 太大需要拆分、技术难度超出能力 |
| **Token 消耗异常高** | 单 session > 500K tokens | 低 | Agent 在调试循环中、上下文膨胀 |
| **Harness 崩溃退出** | tmux session 不存在 + 进度未 100% | 高 | Python 异常、API 错误未被 catch |

### 5.4 Intervene（干预）— 具体操作

#### 干预 Level 1：软干预（不停 Harness）

```bash
# 修改 issue 描述，让下个 session 看到更清晰的指令
# 直接编辑 .issue_store/issues.json 中对应 issue 的 description
# 或通过 comments.json 添加补充说明

python3 -c "
import json
with open('~/projects/{project}/.issue_store/comments.json', 'r') as f:
    comments = json.load(f)
comments.setdefault('ISSUE-{N}', []).append({
    'id': 'jonathan-hint-001',
    'body': '⚠️ Jonathan 补充说明：这个 issue 需要先完成 X，然后再做 Y。注意不要用 Z 方案。',
    'createdAt': '2026-02-26T12:00:00Z'
})
with open('~/projects/{project}/.issue_store/comments.json', 'w') as f:
    json.dump(comments, f, indent=2)
"
```

#### 干预 Level 2：Session 级干预

```bash
# 停止当前 session
tmux send-keys -t harness-{project} C-c

# 等待优雅退出
sleep 5

# 重启（Harness 会从 .issue_store 恢复，不会从头开始）
tmux send-keys -t harness-{project} \
  "python autonomous_agent_demo.py --project-dir ~/projects/{project}" Enter
```

#### 干预 Level 3：Prompt 级干预（Ralph Loop V2）

```bash
# 修改 coding_prompt.md 加入项目特定指令
# 这影响所有后续 session

cat >> ~/harness-openai/prompts/coding_prompt_override.md << 'EOF'

## 项目特定指令（Jonathan 注入）

- 这个项目使用 SQLite，不要尝试安装或配置 PostgreSQL
- 所有 API 返回格式必须是 { success: boolean, data: any, error?: string }
- 前端组件使用 shadcn/ui，不要手写 UI 组件
EOF
```

#### 干预 Level 4：代码级干预（最后手段）

```bash
# 回退有问题的 commit
cd ~/projects/{project}
git revert HEAD --no-edit

# 或切到干净状态
git stash

# 然后重启 Harness
```

### 5.5 Evaluate（评估）— Session 产出评分

Jonathan 应该在每个 session 结束后评估该 session 的质量：

```
Session 评估模板：

Session #: [N]
持续时间: [从 log 中提取]
Token 消耗: [input/output]

产出：
  - Issue 完成数: [N] 个
  - Git commit 数: [N] 个
  - 代码行数变化: +[N] / -[N]

质量指标：
  - [ ] Issue 完成的功能可以工作（通过浏览器/命令行验证）
  - [ ] 代码有适当的错误处理
  - [ ] 没有引入明显的 regression
  - [ ] commit message 有意义

问题（如有）：
  - [描述遇到的问题和采取的干预措施]

经验教训：
  - [记录到 memory 中，供未来 session 参考]

评分: [1-5]
  5 = 所有 issue 完成且质量好
  4 = issue 完成但有小问题
  3 = 部分完成，需要后续修复
  2 = 几乎没有有效产出
  1 = 空跑或造成 regression
```

### 5.6 MDIE Loop 的触发机制

Jonathan 在以下时机执行 MDIE Loop：

| 触发条件 | 执行内容 |
|----------|----------|
| Harness 启动后 10 分钟 | M（检查是否正常启动） |
| 每 30 分钟 | M（进度快照） |
| 检测到 tmux session 结束 | M + D + E（完整评估） |
| 壮爸通过 Telegram 问进度 | M + 汇报 |
| 评估发现问题（评分 ≤ 2） | D + I（诊断 + 干预） |
| 所有 issue 完成 | E + 收尾流程 |

---

## 六、动态 Prompt 修改（Ralph Loop V2）

### 6.1 概念

传统方式：spec 写好 → Harness 跑 → 出了问题手动调试。
Ralph Loop V2：**Jonathan 根据 session 评估结果，自动修改 coding_prompt.md 来纠正 Agent 行为**。

### 6.2 实现方式

Harness 的 prompt 加载流程（见 agent.py）：

```python
prompt = get_coding_prompt()       # 读取 coding_prompt.md
prompt += branch_hint              # 分支提醒
prompt += issue_info               # Issue 列表
```

**Jonathan 可以在 `coding_prompt.md` 末尾追加项目特定指令**，这些指令会在每个新 session 生效。

### 6.3 常见纠正模式

| 问题 | 纠正 prompt |
|------|-------------|
| Agent 总是跳过浏览器测试 | "**强制规则**：每个 issue 完成后必须 browser_screenshot 验证，截图保存到 screenshots/" |
| Agent 用了错误的技术栈 | "**禁止使用 Express.js**。本项目使用 Fastify。如果你发现 Express 代码，先迁移到 Fastify。" |
| Agent 的 CSS 很丑 | "**设计规范**：所有页面使用 12 栏 grid 布局，主色调 #3B82F6，圆角 8px，间距倍数 4px。" |
| Agent 没有处理错误 | "**错误处理规范**：所有 API 调用必须 try-catch，失败时显示 toast 通知用户。" |
| Agent commit message 无意义 | "**Git 规范**：commit message 必须包含 issue 编号，格式 'feat(RUN-XX): 具体描述'" |

### 6.4 自动化纠正流程

```
Jonathan 评估 Session N：评分 2，Agent 跳过了所有浏览器测试
    │
    ▼
Jonathan 分析原因：coding_prompt.md 中浏览器测试步骤不够强制
    │
    ▼
Jonathan 在 coding_prompt.md 末尾追加：
  "CRITICAL: 你必须在每个 issue 完成后执行 browser_navigate + browser_screenshot。
   如果你跳过浏览器验证就标记 Done，这个 issue 会被 Jonathan 重新打开。"
    │
    ▼
Jonathan 重启 Harness → Session N+1 使用更新后的 prompt
    │
    ▼
Jonathan 评估 Session N+1：Agent 执行了浏览器测试 → 评分 4
    │
    ▼
Jonathan 记录经验："浏览器测试跳过问题已通过 prompt 强化解决"
```

---

## 七、多 Worktree 对比模式（高级）

> 壮爸表达了兴趣："多worktree cross-comparison with different models"

### 7.1 概念

同一个 app_spec，用不同的 model（或不同的 coding_prompt）生成两个独立的项目，然后 Jonathan 对比它们的质量，选择更好的。

### 7.2 实现方式

```bash
# 创建两个 worktree
mkdir -p ~/projects/{project}-variant-a
mkdir -p ~/projects/{project}-variant-b

# 复制同一个 app_spec
cp ~/harness-openai/prompts/app_spec.txt ~/projects/{project}-variant-a/
cp ~/harness-openai/prompts/app_spec.txt ~/projects/{project}-variant-b/

# 变体 A：用 codex-high model
OPENAI_MODEL="gpt-5.3-codex-high" \
python autonomous_agent_demo.py --project-dir ~/projects/{project}-variant-a

# 完成后，变体 B：用 codex-low model（成本更低）
OPENAI_MODEL="gpt-5.3-codex-low" \
python autonomous_agent_demo.py --project-dir ~/projects/{project}-variant-b
```

### 7.3 对比维度

| 维度 | 检查方法 |
|------|----------|
| 功能完成度 | 对比 .issue_store 中 Done 的百分比 |
| 代码量 | `find . -name "*.ts" -o -name "*.tsx" \| xargs wc -l` |
| Token 消耗 | 对比 logs 中的 token 统计 |
| Session 数量 | 对比完成所需的 session 数 |
| 代码质量 | `npm run lint` 错误数、TypeScript 类型错误数 |
| 浏览器测试 | 两个版本分别访问，截图对比 |

### 7.4 当前限制

- **顺序执行**：只能先跑完 A 再跑 B（token 每日上限）
- **时间成本高**：两个完整项目可能各需 1-3 天
- **适用场景**：只有重要项目值得这样做，日常小工具不值得

---

## 八、Jonathan 服务器文件更新计划

### 8.1 需要更新的文件

| 文件 | 位置 | 更新内容 |
|------|------|----------|
| **TOOLS.md** | Jonathan 服务器 openclaw 配置 | 添加 harness 启动/监控/干预命令 |
| **AGENTS.md** | Jonathan 服务器 openclaw 配置 | 添加 "Harness 编排者" 角色描述 |
| **MEMORY.md** | Jonathan 服务器 openclaw 配置 | 添加 harness 使用经验区域 |

### 8.2 TOOLS.md 新增内容

```markdown
## Harness 编排命令

### 启动 Harness
```bash
tmux new-session -d -s harness-{project} \
  -c ~/projects/harness-openai \
  "OPENAI_MODEL={model} python autonomous_agent_demo.py \
   --project-dir ~/projects/{project}"
```

### 监控进度
```bash
cat ~/projects/{project}/.issue_store/issues.json | python3 -c "
import json, sys; issues = json.load(sys.stdin)
total = len(issues)
done = sum(1 for i in issues.values() if i.get('state',{}).get('name')=='Done')
print(f'进度: {done}/{total} ({done/total*100:.0f}%)')
"
```

### 检查存活
```bash
tmux has-session -t harness-{project} 2>/dev/null && echo "运行中" || echo "已停止"
```

### 读取日志
```bash
tail -30 ~/projects/{project}/logs/all_sessions.txt
```

### 停止 Harness
```bash
tmux send-keys -t harness-{project} C-c
```

### 重启 Harness（从上次中断处继续）
```bash
tmux send-keys -t harness-{project} \
  "python autonomous_agent_demo.py --project-dir ~/projects/{project}" Enter
```
```

### 8.3 AGENTS.md 新增角色

```markdown
## Harness 编排者

当壮爸要求开发一个新项目时，你的角色是 **Harness 编排者**：

1. **理解需求**：通过对话了解壮爸想要什么
2. **生成 app_spec.txt**：遵循 app_spec 最佳实践模板
3. **启动 Harness**：在 tmux 中运行 autonomous_agent_demo.py
4. **监控-调试-评估**：定期检查进度，发现问题主动干预
5. **通过 Telegram 汇报**：让壮爸随时了解进展
6. **项目收尾**：完成后 git push + PR

你**不需要**自己写代码。你的价值在于：
- 写出高质量的 app_spec
- 在 Harness 卡住时正确诊断和干预
- 持续学习什么样的 spec/prompt 能产出更好的代码
```

---

## 九、分步实施时间线

| 步骤 | 内容 | 预计时间 | 前置条件 |
|------|------|----------|----------|
| **Step 1** | 将 Harness 代码复制到 Jonathan 服务器 | 10 分钟 | 壮爸执行 scp |
| **Step 2** | 安装 Python 依赖 (pip install) | 5 分钟 | Step 1 |
| **Step 3** | 配置环境变量 (.env) | 5 分钟 | Step 2 |
| **Step 4** | 极简测试验证 Harness 能启动 | 30 分钟 | Step 3 |
| **Step 5** | 更新 Jonathan 的 TOOLS.md | 15 分钟 | Step 4 成功 |
| **Step 6** | 更新 Jonathan 的 AGENTS.md | 15 分钟 | Step 5 |
| **Step 7** | 更新 Jonathan 的 MEMORY.md | 10 分钟 | Step 6 |
| **Step 8** | 端到端测试：壮爸 → Jonathan → Harness → 完成 | 1-2 小时 | Step 7 |
| **Step 9** | 记录经验，调整参数 | 持续 | Step 8 |

**总计首次部署时间：~2-3 小时**（包括测试）

---

## 十、风险与缓解

| 风险 | 概率 | 影响 | 缓解措施 |
|------|------|------|----------|
| **right.codes 每日 token 上限** | 确定 | 中 | 单线程顺序执行，每 session 结束检查剩余额度 |
| **Harness 在服务器上行为不同** | 中 | 中 | Step 4 验证测试，确认基本功能正常 |
| **浏览器测试占用过多 RAM** | 低 | 中 | 先不装 Chromium，纯编码模式，需要时再加 |
| **Jonathan 的工具调用鲁棒性 (7/10)** | 中 | 高 | MDIE Loop 中的干预机制兜底 |
| **app_spec 质量不够好** | 中 | 高 | 遵循最佳实践模板 + 壮爸确认流程 |
| **Harness 和 Jonathan 竞争 API 配额** | 低 | 中 | Harness 运行时 Jonathan 只做监控（不消耗 LLM token） |

---

## 十一、成功标准

### 最小可行成功（MVP）
- [ ] Harness 在 Jonathan 服务器上能启动并完成至少 1 个 session
- [ ] Jonathan 能通过 exec 工具启动/监控/停止 Harness
- [ ] Jonathan 能通过 Telegram 向壮爸汇报进度

### 完整成功
- [ ] Jonathan 能从壮爸的 Telegram 消息生成 app_spec.txt
- [ ] Jonathan 能运行完整的 MDIE Loop（监控 → 诊断 → 干预 → 评估）
- [ ] 一个真实项目从 spec 到完成全部由 Jonathan 编排
- [ ] Jonathan 能根据评估结果动态修改 coding_prompt.md

### 进阶成功
- [ ] 多 worktree 对比模式工作
- [ ] Jonathan 能主动发现并修复 Harness session 的问题
- [ ] 积累 3+ 个项目的经验数据，形成持续改进循环

---

## 十二、附录：Harness 技术要点速查

> 供执行者快速了解 Harness 的工作原理，不需要读完 686 行的 HARNESS_DEEP_DIVE.md。

### 核心文件

| 文件 | 行数 | 职责 |
|------|------|------|
| `autonomous_agent_demo.py` | ~100 | 入口：CLI 参数 + 启动主循环 |
| `agent.py` | ~617 | 多 session 循环、prompt 构造、日志 |
| `client.py` | ~1226 | API 调用、工具执行、终止检测 |
| `tools.py` | ~484 | Read/Write/Edit/Bash/Glob/Grep |
| `browser_tools.py` | ~586 | 18 个 Playwright 浏览器工具 |
| `local_issue_store.py` | ~500 | Issue JSON 存储 + 3 个 Agent 工具 |
| `security.py` | ~360 | Bash 命令白名单 |

### 工作流

```
Session 1（初始化）:
  app_spec.txt → 创建 ~100 个 Issue → init.sh → git init → .project.json

Session 2+（编码）:
  12 步循环: 了解环境 → 选 Issue → 认领 → 实现 → 浏览器验证 → commit → 更新 META

Session 间衔接:
  检查 has_actionable_issues() → True → 新 session / False → PROJECT COMPLETE
```

### 5 层终止检测

1. **END_SESSION token** — Agent 显式声明结束（门控：必须写过代码）
2. **Idle Loop** — 5 轮空闲命令 → 强制结束，10 轮 → 硬退出
3. **Tail Injection** — ≤80 turns 提醒 commit，≤50 禁止新 issue，≤20 强制收尾
4. **No-Tool-Turn** — 连续多轮不调工具 → 递进催促
5. **Max Turns** — 300 轮硬限制

### Issue Store 结构

```
.issue_store/
├── issues.json    # { "RUN-1": { identifier, title, description, priority, state, ... } }
├── comments.json  # { "RUN-1": [ { id, body, createdAt } ] }
└── meta.json      # { prefix, next_id, created_at, schema_version }
```

### 环境变量

| 变量 | 必需 | 说明 |
|------|------|------|
| `OPENAI_API_KEY` | ✅ | right.codes API key |
| `OPENAI_BASE_URL` | ✅ | `https://right.codes/codex/v1` |
| `OPENAI_MODEL` | ❌ | 默认 `gpt-5.2-codex-high` |
| `BROWSER_HEADLESS` | ❌ | 默认 `true` |
| `LOG_NATURAL_ONLY` | ❌ | 默认 `false`，建议 `true` |

---

*本文档是 Jonathan + Harness 集成的完整构建计划。任何新的 context window 读完本文档后，应该能直接动手实施 Step 1-9。*

*相关文档：`2026-02-26_agent-swarm-analysis.md`（背景分析）、`2026-02-26_harness-integration-insights.md`（架构洞察）*
