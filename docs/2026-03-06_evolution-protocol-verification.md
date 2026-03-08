---
title: Evolution Protocol 验证指南
date: 2026-03-06
tags: [evolution, verification, audit]
depends_on: []
status: current
last_updated: 2026-03-06
---

# Evolution Protocol 验证指南

部署日期：2026-03-06。本文档用于在部署一段时间后，系统性验证进化协议是否被各 agent 正确执行。

## 何时验证

- **首次验证**：部署后 3 天（2026-03-09）
- **二次验证**：部署后 7 天（2026-03-13）
- **长期**：纳入老大爷每周审计，不再需要手动验证

## 验证清单

### 1. 协议14：自检信号区块

**预期**：每个 agent 的 MEMORY.md 末尾都有 `## 自检信号` 区块，且内容非模板（说明 agent 真正在维护它）。

```bash
# 一键检查所有 agent 的自检信号状态
ssh -i ~/.ssh/id_ed25519 zhuangba@192.168.0.18 '
for ws in workspace workspace-gatekeeper workspace-nurse workspace-teacher workspace-mail-assistant workspace-naonao; do
  mem="$HOME/.openclaw/$ws/MEMORY.md"
  name=$(echo $ws | sed "s/workspace-//;s/workspace/main/")
  if grep -q "^## 自检信号" "$mem" 2>/dev/null; then
    echo "✓ $name — 自检信号存在:"
    sed -n "/^## 自检信号/,\$p" "$mem" | head -5
  else
    echo "✗ $name — 自检信号缺失"
  fi
  echo ""
done
'
```

**评判标准**：

| 结果 | 评价 | 行动 |
|------|------|------|
| 6/6 有自检信号，内容非模板 | 协议14完全生效 | 无需干预 |
| 4-5/6 有 | 部分生效，检查缺失 agent 是否有过 session | 对未触发的 agent 发一条消息触发 session |
| ≤3/6 有 | 协议措辞可能不够强 | 检查 SHARED_RULES 措辞，考虑从"must"升级为更具体的执行指令 |
| 有但内容全是模板默认值 | agent 只是复制了格式，没有真正维护 | 在 SHARED_RULES 中补充示例，明确"已知问题"应记录什么 |

### 2. 协议15：弹性容量

**预期**：超过 30 行的 agent（当前 main=55, teacher=52）应该开始 Level 1 拆分。

```bash
# 检查各 agent 记忆行数和是否已拆分
ssh -i ~/.ssh/id_ed25519 zhuangba@192.168.0.18 '
for ws in workspace workspace-gatekeeper workspace-nurse workspace-teacher workspace-mail-assistant workspace-naonao; do
  mem="$HOME/.openclaw/$ws/MEMORY.md"
  memdir="$HOME/.openclaw/$ws/memory"
  name=$(echo $ws | sed "s/workspace-//;s/workspace/main/")
  lines=$(wc -l < "$mem" 2>/dev/null || echo 0)
  has_memdir=$([ -d "$memdir" ] && echo "YES ($(ls "$memdir"/*.md 2>/dev/null | wc -l) files)" || echo "NO")
  echo "$name: ${lines} lines, memory/ dir: $has_memdir"
done
'
```

**评判标准**：

| 结果 | 评价 | 行动 |
|------|------|------|
| >30 行的 agent 已创建 memory/ 目录并拆分 | 协议15生效 | 无需干预 |
| >30 行但未拆分 | agent 未响应弹性容量规则 | 检查措辞；或在下次评估时提醒该 agent |
| 所有 agent ≤30 行 | 还没到触发阈值 | 正常，继续观察 |

### 3. 协议16：错误记录

**预期**：自检信号的"已知问题"和"近期失败"字段应反映真实错误，不是空的"无"。

```bash
# 交叉验证：对比自检信号 vs 实际 session 错误
ssh -i ~/.ssh/id_ed25519 zhuangba@192.168.0.18 '
echo "=== 自检信号中的已知问题 ==="
for ws in workspace workspace-gatekeeper workspace-nurse workspace-teacher workspace-mail-assistant workspace-naonao; do
  mem="$HOME/.openclaw/$ws/MEMORY.md"
  name=$(echo $ws | sed "s/workspace-//;s/workspace/main/")
  problems=$(sed -n "/^## 自检信号/,\$p" "$mem" 2>/dev/null | grep "已知问题" || echo "(无自检信号)")
  echo "$name: $problems"
done

echo ""
echo "=== 实际 session 错误（近 3 天） ==="
for agent_id in main gatekeeper nurse teacher mail-assistant naonao; do
  session_dir="$HOME/.openclaw/agents/$agent_id/sessions"
  if [ -d "$session_dir" ]; then
    error_files=$(find "$session_dir" -name "*.jsonl" -mtime -3 -exec grep -l "\"error\"" {} \; 2>/dev/null | wc -l)
    echo "$agent_id: ${error_files} sessions with errors (3 days)"
  else
    echo "$agent_id: no session dir"
  fi
done
'
```

**评判标准**：

| 自检信号说"无" + 日志有错误 | 盲区 — agent 没有记录错误 | SHARED_RULES 协议16 措辞需加强 |
|---|---|---|
| 自检信号列出了问题 + 日志吻合 | 协议16完全生效 | 最佳状态 |
| 自检信号列出了问题 + 问题已修复但未清除 | agent 没有执行"修复后清除" | 提醒或加强措辞 |

### 4. 老大爷审计链路

**预期**：08:30 cron 生成 daily-audit.txt → 老大爷 heartbeat 读取 → 分析 → 写 pending-review.md 或告警。

```bash
# 检查审计链路是否运转
ssh -i ~/.ssh/id_ed25519 zhuangba@192.168.0.18 '
echo "=== Cron 日志（最近 3 天） ==="
tail -5 ~/.openclaw/workspace-gatekeeper/logs/daily-audit-cron.log 2>/dev/null || echo "无日志"

echo ""
echo "=== daily-audit 文件记录 ==="
ls -la ~/.openclaw/shared/daily-audit*.txt 2>/dev/null || echo "无审计文件"

echo ""
echo "=== pending-review.md 状态 ==="
if [ -f ~/.openclaw/shared/pending-review.md ]; then
  head -10 ~/.openclaw/shared/pending-review.md
else
  echo "不存在（老大爷可能还未处理，或无异常）"
fi
'
```

**评判标准**：

| 结果 | 评价 | 行动 |
|------|------|------|
| daily-audit.txt 每天生成 + done 文件存在 | 采集+审计链路完整 | 无需干预 |
| daily-audit.txt 生成但无 done 文件 | 老大爷没有处理审计报告 | 检查老大爷 SOUL.md 审计指令是否足够具体 |
| daily-audit.txt 未生成 | cron 脚本问题 | 检查 cron 日志 |

### 5. 环形监督闭环

**预期**：老大爷审计其他 agent，Jonathan 审计老大爷。

```bash
# 检查 Jonathan 是否在观察老大爷
ssh -i ~/.ssh/id_ed25519 zhuangba@192.168.0.18 '
echo "=== Jonathan MEMORY 中是否提到老大爷审计 ==="
grep -i "gatekeeper\|老大爷\|审计\|audit" ~/.openclaw/workspace/MEMORY.md 2>/dev/null || echo "未提及"

echo ""
echo "=== 老大爷最近 session 是否包含审计行为 ==="
latest=$(ls -t ~/.openclaw/agents/gatekeeper/sessions/*.jsonl 2>/dev/null | head -1)
if [ -n "$latest" ]; then
  grep -c "daily-audit\|pending-review\|自检信号" "$latest" 2>/dev/null || echo "0 matches"
else
  echo "无 session"
fi
'
```

## 验证后的行动矩阵

| 总体评价 | 行动 |
|---------|------|
| **全部生效** | 协议成功，记录到评估文档，进入 Phase 3（交叉评估质量提升） |
| **部分生效（≥4/6 agent 响应）** | 分析未响应 agent 的原因（无 session？措辞不够强？），微调 SHARED_RULES |
| **大部分未生效（≤3/6）** | 协议措辞需要根本性修改——从"must maintain"改为更具体的执行步骤和触发时机 |
| **审计链路断裂** | 优先修复 cron → 老大爷处理链路，这是整个体系的心跳 |

## 措辞调优参考

基于 D1-D9 的核心经验："按需"=跳过，"必须"=执行。如果协议14未被执行，考虑将 SHARED_RULES 措辞从：

```
Every agent must maintain a 自检信号 section...
```

升级为：

```
After completing any task, your FINAL action before ending the session is:
1. Open MEMORY.md
2. Update the 自检信号 section at the bottom
3. If no 自检信号 section exists, create it now
This is not optional. Skipping this step is a protocol violation.
```

---

_本文档为一次性验证指南。当进化协议稳定运转后，验证职能由老大爷每周审计自动覆盖，本文档可归档。_
