---
title: Token Usage Analysis
date: 2026-02-25
tags: [tokens, cost, performance, optimization]
depends_on: [2026-02-24_setup-summary.md]
status: current
last_updated: 2026-02-25
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

## System Prompt Overhead（不变）

Total system prompt: **25,405 chars ≈ 5,500 tokens/call**

| Component | Chars | % |
|-----------|------:|--:|
| Tool schemas (21 tools) | 14,590 | 57.4% |
| AGENTS.md | 7,804 | 30.7% |
| Skills descriptions (5) | 2,450 | 9.6% |
| 其余（SOUL/TOOLS/IDENTITY/USER/HEARTBEAT/BOOTSTRAP） | 5,237 | 20.6% |

> AGENTS.md + Tool schemas 占系统 prompt 的 88%。

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

- 精简 AGENTS.md（去掉不用的群聊规则等）可节省 ~3K tokens/call
- 保持长会话以利用 cache（但定期 `/new` 避免 context 溢出）
- 修复 message 工具参数问题，消除无效重试
- 对简单任务限制工具调用轮数
