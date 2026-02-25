---
title: Token Usage Analysis
date: 2026-02-24
session_id: bb6cb8ff-546a-4a70-bace-c315da01d2e8
tags: [tokens, cost, performance, optimization]
depends_on: [2026-02-24_setup-summary.md]
status: current
last_updated: 2026-02-25
---

# Token Usage Analysis - First Session

## Session Overview
| Metric | Value |
|--------|-------|
| Duration | 17:37 - 22:16 CST (4h39m) |
| User messages | 6 |
| API calls | 26 (verified by both Dashboard and API provider) |
| Total tokens | **326.6K** |
| Input tokens | 259.2K (79.4%) |
| Output tokens | 11.3K (3.5%) |
| Cache read | 56.1K (17.2%) |
| Cache write | 0 |
| Total cost | ~$0.55 (套餐扣费) |
| Errors | 0 |
| Cache hit rate | 17.8% |

## System Prompt Overhead (per API call)
Total system prompt: **25,405 chars ≈ 5,500 tokens**

| Component | Chars | % of System Prompt |
|-----------|------:|-------------------:|
| AGENTS.md | 7,804 | 30.7% |
| Tool schemas (21 tools) | 14,590 | 57.4% |
| Skills descriptions (5) | 2,450 | 9.6% |
| SOUL.md | 1,664 | 6.5% |
| BOOTSTRAP.md | 1,449 | 5.7% |
| TOOLS.md | 850 | 3.3% |
| IDENTITY.md | 633 | 2.5% |
| USER.md | 474 | 1.9% |
| HEARTBEAT.md | 167 | 0.7% |
| Tool list | 1,724 | 6.8% |

**AGENTS.md + Tool schemas account for 88% of system prompt.**

## Per-Message Breakdown

### Msg 1: "你好" — 10.3K tokens, 1 API call, 15s
- Simple greeting, no tools used
- System prompt dominated (76% of input)

### Msg 2: "你叫Jonathan 我叫zhuangba" — 73.7K tokens, 7 API calls, 45s
- read IDENTITY.md → write IDENTITY.md → read USER.md → write USER.md → git status → git commit (fail) → reply
- Most expensive message: 7 sequential API calls, each resending full context

### Msg 3: "我配置好了git环境" — 61.1K tokens, 5 API calls, 48s
- git commit → edit BOOTSTRAP.md → git rm (fail) → ls+status → reply
- Largest single output: 1,365 tokens (final reply)

### Msg 4: "打开user.md文件夹" — 25.9K tokens, 2 API calls, 18s
- xdg-open (failed silently) → false success reply
- 0 cache hits (gateway had restarted)

### Msg 5: "确认命令生效了吗？" — 68.3K tokens, 5 API calls, 39s
- DISPLAY check → X11/sessions → find file managers → thunar+DISPLAY=:1 → admit error + fix
- Demonstrated self-debugging capability

### Msg 6: "记下来了吗？" — 87.2K tokens, 6 API calls, 48s
- memory_search → date → create memory file → read verify → git commit → reply
- Highest per-call input (14.5K) due to longest conversation history

## Key Insights

1. **每条消息平均 54.4K tokens** — "你好" 就花了 10K
2. **99% 的 token 是"思考"而非"说话"** — Output 仅占 3.5%
3. **多步工具调用是主要成本** — 每次工具调用都重发完整上下文
4. **Cache hit 偏低 (17.8%)** — Gateway 重启导致缓存失效，理想应 >50%
5. **对话越长单次调用越贵** — 从 ~8K/call 增长到 ~14.5K/call
6. **API 出口 IP**: `188.253.113.70` (mihomo 代理出口)

## Optimization Suggestions
- 精简 AGENTS.md（去掉不用的群聊规则、心跳详细说明等）可节省 ~3K tokens/call
- 避免频繁重启 Gateway 以保持 cache
- 对简单任务考虑限制工具调用轮数
- 长对话定期 `/new` 重置会话

---

## Day 2 Session (2026-02-25) — 跨天延续

| Metric | Day 1 | Day 2 | Notes |
|--------|-------|-------|-------|
| Session ID | bb6cb8ff... | 576c53d4...（同一会话延续） | Day 2 是 /reset 后的新会话 |
| Duration | 4h39m | ~19h（00:11 - 19:31） | 跨天同一会话 |
| Telegram messages | 6 user msgs | ~76 条发送 | 活跃度大幅提升 |
| Input tokens | 259.2K | 144,099 | |
| Output tokens | 11.3K | 736 | 极低，多数输出走 Telegram |
| Cache read | 56.1K | 131,584 | cache 利用率大幅提升 |
| Total tokens | 326.6K | 137,942 | 更高效（持续会话 cache 命中） |
| Compaction count | - | 0 | 未触发压缩 |
| Context window | 1,000,000 | 1,000,000 | |
| Errors | 0 | 43 | 40x message 参数错误 + 3x timeout |

### Day 2 Key Observations

- **Cache 命中大幅提升**：131K cache read（占 input 的 91%），因为是长时间同一会话
- **Output 异常低（736 tokens）**：因为大多数回复通过 Telegram message 工具发送，不计入 output
- **错误率高**：43 次错误（40 次 message 参数错误 + 3 次 10 分钟超时）
- **效率提升**：总 token 降至 138K（Day 1 的 42%），得益于高 cache 命中
