# D5 质量监控优化 — 接续 Prompt

先运行 `/jonathan-context` 加载背景，然后执行以下任务。

## 背景

D5 (2026-02-28) 完成了 Harness 监控层的进度监控验证和修复（sleep 上限、汇报触发、超时预警、HEARTBEAT 重写等）。进度层已经闭环。

但 D5 同时暴露了一个更大的问题：**Jonathan 的质量监控完全空白**。MDIE.md 里写了 Diagnose 6 种模式和 Intervene 4 个级别，D5 全程一次都没触发。Jonathan 只会数 Done/Total，从不检查代码能不能跑、有没有 regression、项目目录是不是垃圾堆。

## D5 质量层面的具体发现

以 guess-number 项目（~/projects/guess-number/）为例：

| 观察 | Jonathan 做了？ |
|------|----------------|
| 12/12 issues Done | ✅ 数了 |
| main.py 67 行，结构清晰，代码质量好 | ❌ 没看过代码 |
| 项目根目录散落 20+ 个 .png 截图（Harness 验证截图） | ❌ 没发现 |
| verify_issue8.js 被创建又删除（commit 6→8，regression 模式） | ❌ 没注意 |
| 没有 logs/ 目录（MDIE 的 `tail -30 logs/` 永远会失败） | ❌ 没发现 |
| 没有自动化测试 | ❌ 没检查 |
| 代码是否能跑 | ❌ **从未执行 `python3 main.py`** |
| Intervene 4 级干预工具 | ❌ 一次都没用过 |

根因与进度监控问题相同：MDIE 里质量检查不是"必须"，所以 Jonathan 不做。

## 需要修改的文件

### 1. MDIE.md（服务器 ~/.openclaw/workspace/MDIE.md）

在 Monitor 段落之后、Diagnose 之前，新增一个 **[Q] Quality Check** 段落，标注为"每次 Monitor 后必须执行"。内容包括：

- **代码可运行性检查**：CLI 项目 `echo "50" | python3 main.py`，Web 项目 `curl localhost:{port}`。只要不 crash 就 pass。
- **最近 commit 变更审查**：`git diff HEAD~1 --stat`。检测文件被删除、大量行被删、同一文件反复改等 regression 信号。
- **项目卫生检查**：根目录 .png/.tmp/.log 文件数量，超过 5 个则用 Level 1 干预（comments.json 告诉 Harness 放到子目录）。
- **无限 session 检测**：所有 issue Done + tmux 仍在运行超过 5 分钟 → 直接 tmux send-keys C-c 停掉。

措辞要求：关键步骤用"必须"，不用"按需"或"可选"。D5 实证 Jonathan 对措辞字面执行。

### 2. coding_prompt.md（服务器 ~/projects/harness-openai/prompts/coding_prompt.md）

在文件**末尾**追加项目卫生规则（影响所有后续 Harness 项目）：

```
### 项目卫生规则
- 截图文件必须放在 screenshots/ 子目录，禁止散落在项目根目录
- 每个 issue 完成后，运行一次主入口确认不 crash
- 临时验证脚本（verify_*.js 等）用完即删，不要留在 commit 中
```

### 3. Jonathan MEMORY.md（服务器 ~/.openclaw/workspace/MEMORY.md）

追加一条：

```
7. **质量检查经验**
   监控 Harness 时不能只数 Done/Total。每次 Monitor 后必须执行质量检查：运行代码确认不 crash、审查最近 commit 变更、检查项目目录卫生。详见 MDIE.md [Q] 段落。
```

### 4. 本地记忆文件更新

按 `/jonathan-update-memory` 标准更新：
- evaluation.md：Capability Boundaries 增加"Cannot: 主动质量检查"→ 改完后如果验证通过则更新为 Can
- harness-deployment.md：记录质量监控修复
- README.md：TL;DR 更新

## 执行步骤

1. SSH 到服务器修改 MDIE.md、coding_prompt.md、MEMORY.md
2. 验证修改正确（grep 关键行）
3. 更新本地记忆文件
4. git commit + push

## SSH 连接

连接信息在 CLAUDE.local.md 中自动加载。

## 注意事项

- 600s agent run timeout 是平台 embedded 限制，不可配置
- 服务器在中国大陆，pip 用 mirrors.aliyun.com
- SSH 长命令中 Python f-string 引号嵌套容易出错，用 heredoc（`<< 'PYEOF'`）更稳定
- Jonathan 对手册措辞字面执行：**"按需"=跳过，"必须"=执行**，写手册时切记
