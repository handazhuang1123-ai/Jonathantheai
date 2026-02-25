---
title: Token Usage Analysis
date: 2026-02-25
tags: [tokens, cost, performance, optimization]
depends_on: [2026-02-24_setup-summary.md]
status: current
last_updated: 2026-02-26
supersedes: archive/2026-02-24_token-analysis.md
---

# Token Usage Analysis

## Day-over-Day Summary

| Metric | Day 1 (2/24) | Day 2 (2/25) | Δ |
|--------|-------------|-------------|---|
| Session ID | bb6cb8ff... | 576c53d4... | /reset 后新会话 |
| Duration | 4h39m | ~19h | 跨天同一会话 |
| User messages | 6 | ~76 条 Telegram 发送 | 活跃度大幅提升 |
| Total tokens | **326.6K** | **137.9K** | -58%（更高效） |
| Input tokens | 259.2K | 144,099 | |
| Output tokens | 11.3K | 736 | 极低（回复走 Telegram） |
| Cache read | 56.1K (17.8%) | 131,584 (**91%**) | 持续会话 cache 命中 |
| Errors | 0 | **43** | 40x message 参数 + 3x timeout |
| Compaction | - | 0 | 未触发压缩 |
| Context window | 1,000,000 | 1,000,000 | |

## Day 2 Key Observations

- **Cache 效率质变**：91% cache read vs Day 1 的 17.8%，长会话复用效果显著
- **Output 统计失真**：736 tokens output 不反映真实回复量，因为大多数回复通过 message 工具发送（不计入 output）
- **错误集中爆发**：43 次错误全部集中在 message 工具参数兼容问题和超时，非模型能力问题
- **总量效率提升**：虽然活跃时间更长（19h vs 4.6h），总 token 反而降低 58%

## System Prompt Overhead

> 2/26 壮爸直接精简了 AGENTS.md，system prompt 大幅缩减。

| Component | 精简前 | 精简后 | Δ |
|-----------|------:|------:|--:|
| AGENTS.md | 7,804 | 3,143 | **-60%** |
| Tool schemas (21 tools) | 14,590 | 14,590 | 不变（需平台侧优化） |
| Skills descriptions (5) | 2,450 | 2,450 | 不变 |
| TOOLS.md + 其余 | ~5,237 | ~5,293 | +56（TOOLS.md 加入实际内容） |
| **Total** | **~25,405** | **~20,800** | **-18%** |

> Tool schemas 仍是最大头（70%），其中 message 工具一个就占 4,220 chars / 85 参数。这部分需要 OpenClaw 平台侧优化（按需加载 schema），用户无法干预。

## Token 放大效应分析

Day 1 用户实际输入约 52 tokens，消耗 326,600 tokens，**放大比 1:6,280**。原因：

1. **固定税**：每次 API 调用重发系统 prompt（~5,500 tokens），26 次调用 = ~143K（占总量 44%）
2. **累积税**：每次工具调用重发完整对话历史，单次 input 从 8K 增长到 14.5K
3. **错误税**：Day 2 的 40 次 message 参数错误 = 40 次无效上下文重发，浪费 ~20K+ tokens

## Day 1 Per-Message Breakdown（基线参考）

| # | Message | Tokens | API Calls | Key Event |
|---|---------|--------|-----------|-----------|
| 1 | "你好" | 10.3K | 1 | 系统 prompt 占 76% |
| 2 | "你叫Jonathan 我叫zhuangba" | 73.7K | 7 | 最贵：7 次顺序调用 |
| 3 | "我配置好了git环境" | 61.1K | 5 | git rm 失败 |
| 4 | "打开user.md文件夹" | 25.9K | 2 | xdg-open 静默失败 |
| 5 | "确认命令生效了吗？" | 68.3K | 5 | 自我调试 DISPLAY |
| 6 | "记下来了吗？" | 87.2K | 6 | 最高单次 input（14.5K/call） |

## Key Insights（累计）

1. **多步工具调用是主要成本** — 每次调用重发完整上下文
2. **Cache 是最大杠杆** — Day 1: 17.8% → Day 2: 91%，总 token 降 58%
3. **避免 Gateway 重启** — 会导致 cache 失效
4. **长对话不一定更贵** — 如果 cache 命中率高，甚至更省
5. **错误会浪费 token** — 40 次无效 message 调用消耗了不必要的资源

## Optimization Suggestions

- ~~精简 AGENTS.md~~ ✅ 已完成（2/26，7,804 → 3,143 chars）
- ~~修复 message 工具参数问题~~ ✅ 已在 TOOLS.md 记录正确参数
- 保持长会话以利用 cache（但定期 `/reset` 避免 context 膨胀）
- 对简单任务限制工具调用轮数
- **Gateway 管理策略**：`/reset` 是当前最优解 — 清空对话历史控制 context 大小，但不重启 Gateway 进程，cache 不失效。频繁重启 Gateway 会丢 cache（token 翻倍），从不重启会导致 context 膨胀和内存风险（Day 2 峰值 794.7MB）
