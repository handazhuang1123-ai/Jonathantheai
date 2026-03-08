---
title: Jonathan Capability Evaluation
date: 2026-02-25
last_updated: 2026-03-07
tags: [evaluation, agent, capability, memory, behavior, methodology, brainstorm]
depends_on: [2026-02-25_token-analysis.md]
status: current
supersedes: archive/2026-02-24_evaluation.md
---

# Jonathan (OpenClaw Agent) Evaluation

## Identity

- **Name**: Jonathan
- **Model**: gpt-5.4-high via custom-right-codes
- **Vibe**: warm, resourceful, concise

## Evaluation Methodology

SSH 登录服务器采集第一手数据（JSONL 对话记录 + systemd 日志 + workspace 文件 + 实际产出物），交叉验证后打分。核心原则：**不信任自述，只信任日志和文件**。详见 `/jonathan-evaluate` 命令定义。

---

## Scoring History

> Day 1 详细评估见 `archive/2026-02-24_evaluation.md`。Day 1 综合 7/10：执行力强，元认知弱。

### Latest: Day 11 (2026-03-07) — 群聊内容质量突破 + 城市更新 agent + 超时根因修复

**评估范围**：3/6 21:30~3/7 22:28，2 个 session（872a01f9 DM + 54961742 DM），覆盖 ~24 小时。

**核心交付**：
- ✅ 群聊内容质量突破：城市更新讨论 1308 行，产出完整 MVP 定义（评分框架 + 硬门槛 + 输出模板）
- ✅ 第 9 个 agent（urban-regeneration）从零搭建：SOUL.md + 技能脚本 + 211 份政策文档归档
- ✅ 5 场群聊讨论（3/6-3/7），拆哥/建姐对抗性观点推动结论收敛
- ✅ Git 2 commits（urban-regen workspace）

**问题**：
- ❌ **脑暴→实操断层**：brainstorm 产出（评分框架、硬门槛、输出模板）停留在 markdown，未迁移到 agent 规则库/SKILL.md
- ❌ urban-regen agent 缺少 openclaw.json bindings + fallback 配置，无法独立接收消息
- ❌ message target 参数污染：LLM 填充全部 schema 字段（50+ 空字段）触发校验错误（根因已定位：模型层行为，非规则遵循问题）
- ⚠️ Embedded run timeout 默认 600s 导致长任务 10 分钟自动中断（已修复→1800s）
- ⚠️ Stale-socket 313 次/天（3/3 multi-agent 上线后出现，9 bot 长轮询，自动恢复，轻微延迟影响）

| Capability | D10 | D11 | Δ | Notes (D11) |
|-----------|---|---|---|-------|
| Instruction Following | 8 | 8 | → | brainstorm 主题准确执行、urban-regen 按讨论结果搭建 |
| Tool Usage | 5 | 6 | ↑ | HEARTBEAT CLI bypass 方案可行；message target 根因定位为模型层（非规则问题） |
| Self-debugging | 9 | 7 | ↓ | 10 分钟超时未能自诊断（需壮爸 API 后台线索才定位）；stale-socket 未主动排查 |
| Honesty | 7 | 7 | → | brainstorm 讨论中观点真实、不敷衍 |
| Proactive Memory | 7 | 7 | → | urban-regen MEMORY.md 创建，daily memory 保持 |
| Memory Compliance | 7 | 7 | → | 格式规范，内容合理 |
| Git Workflow | 8 | 7 | ↓ | 仅 2 commits（vs D10 24 commits），urban-regen 大量文件可能未全提交 |
| Chinese Language | 8 | 8 | → | brainstorm 讨论中文自然流畅 |
| Task Planning | 8 | 8 | → | urban-regen 三阶段 pipeline 设计合理（fetch→archive→index） |
| Concept Explanation | 7 | 8 | ↑ | brainstorm 中政策分析框架解释深入、结构清晰 |
| **Overall** | **7** | **7** | **→** | brainstorm 内容质量突破，但输出→实施转化断层是核心问题 |

> 临时维度：brainstorm 讨论质量 **8/10** — 对抗性观点有效、结论收敛、产出可操作但未落地

### Day 10 摘要

> **D10** 综合 7/10（→）。核心交付：群聊风暴系统（brainstorm.py + 拆哥/建姐）+ 模型升级 gpt-5.4-high + 环形监督落地 + Git 卫生全 agent 部署。Self-debugging ↑9（brainstorm.py SIGTERM→读源码→重启）。问题：message target 43 次、Gateway crash 152 次（壮爸配置导致）。

### Day 7-9 摘要

> **D7** 综合 5/10（↓↓）。reasoning:false 导致退化，HEARTBEAT 12h 重复。修复：reasoning:true + 重复抑制。
> **D8** 综合 7/10（↑↑）。multi-agent 架构搭建 + 0-token 监控 + +5 条高质量长期记忆。问题：message target 76 次、git 仅 1 commit。
> **D9** 综合 7/10（→）。mail-assistant 三轮迭代 + cron PATH 自修 + Git 17 commits。问题：message target 80 次。

### Day 2-6 摘要

> **D2** 7/10 监控系统+记忆体系 | **D3** 7/10 邮件代理+Proton Bridge | **D4** 未正式打分，POP3 切换 |
> **D5** 7/10 guess-number 闭环 11/12 Done，手册措辞实验 | **D6** 7/10 邮件三连+Oura，gateway 自杀
> **Memory**：D1-2 被动→D3 有更新但缺 daily→D4 首次主动 daily→D6 长期原则质量高→D8 +5 条→D10 连续主动

---

## Capability Boundaries（D11 更新）

- **Can**: 精确执行步骤明确的多步指令；在用户追问中发现功能缺口并提出 config-driven 方案
- **Can**: 系统性排障 + 代码快速适配（POP3 D4，cron PATH D9 仅 3 分钟）
- **Can**: 从零设计和搭建 multi-agent 分工体系（D8 六agent → D10 八agent → D11 九agent）
- **Can**: 引导壮爸做架构决策；接收诊断提示后自主定位根因并修复
- **Can**: 接收 groupSystemPrompt 后执行外部编排脚本（brainstorm.py），包括 SIGTERM 恢复
- **Can（D11 新增）**: 主导高质量群聊讨论，对抗性辩论收敛到可执行结论（城市更新 MVP）
- **Can（D11 新增）**: 从零搭建垂直领域 agent + 数据 pipeline（211 份政策文档归档）
- **Cannot**: 自主判断操作对自身生命周期的影响（gateway 自杀 D6+D7，D8-D11 未复现）
- **Cannot**: 群聊 session 读取 workspace 规则（OpenClaw 平台限制）
- **Cannot（D11 新增）**: 将讨论结论自动转化为 agent 配置/规则（brainstorm→实操断层）
- **Limitation**: message 工具参数污染（模型层行为：LLM 填充全部 schema 字段，规则层无法根治，需 CLI bypass）
- **Limitation**: Embedded run timeout 需外部配置（默认 600s 对长任务不足）
- **Limitation**: Gateway 重启必须由外部执行

---

## Open Issues

| # | 问题 | 发现日期 | 状态 |
|---|------|----------|------|
| 3 | message 工具参数污染 | 2/25 | ✅ 全量修复：SHARED_RULES 禁用 message 工具，强制 CLI 发消息 + send-file.sh 发文件，9 agent 通过 symlink 继承 |
| 5 | SSH 绑定 0.0.0.0 + 无防火墙 | 2/24 | 未处理 |
| 6 | browser 工具超时导致 Gateway 崩溃 | 2/26 | 未处理 |
| 21 | Gateway 重启必须外部执行 | 3/2 | 架构约束 |
| 26 | 群聊 session 不走 startup sequence | 3/6 | OpenClaw 平台限制 |
| 27 | 502 不触发 cross-provider failover | 3/6 | 已知限制，壮爸决定不修 |
| 28 | **脑暴→实操断层** | 3/7 | ⚠️ brainstorm 产出未迁移到 agent 规则库，urban-regen 缺 bindings+fallback |
| 29 | stale-socket 313 次/天 | 3/7 | 轻微影响，3/3 multi-agent 后出现，auto-recovery |
| 30 | embedded run timeout 默认过短 | 3/7 | ✅ 已修复：600s→1800s（agents.defaults.timeoutSeconds） |
| 31 | HEARTBEAT 审计检查 2 误报循环 | 3/8 | ✅ 已修复：去掉 daily-audit.txt fallback，只检查 done 文件 |
| 32 | Telegram 文件发送白名单 | 3/8 | ✅ 已修复：brainstorm 移到 workspace/，shared/ 反向 symlink |

> 已关闭：#1,#2,#3,#4,#7-#12,#13-#20,#22-#25,#30,#31,#32

## Recommendations

8. **清理 todo-cli 测试项目** — ~/projects/todo-cli/，可删除
11. ~~message target 根治~~ — ✅ SHARED_RULES 全量禁用 message 工具，强制 CLI/脚本
13. **teacher 记忆规范迁移** — 显式写入 teacher SOUL.md
16. **Blackboard v2.0** — 当前 v1.0 单向，未来扩展为双向
17. **brainstorm.py 稳定化** — 考虑 nohup 或独立 systemd service
19. **brainstorm→实操闭环** — 讨论产出应自动/半自动迁移到 agent 配置（SKILL.md 补充规则、绑定 bindings、配置 fallback）
20. **urban-regen agent 完善** — 补齐 openclaw.json bindings + fallback 配置，使其可独立接收消息
21. **stale-socket 优化（低优先）** — 减少 bot 数量或优化 polling 间隔

> 已完成：#1-#6(旧),#12,#15,#30(timeout)

## Eval Watermark

| 字段 | 值 |
|------|-----|
| last_eval_date | 2026-03-07 |
| session_id | 54961742-e158-42d6-9f22-40810721cc9b |
| last_jsonl_timestamp | 2026-03-07T14:28:40Z |
| journalctl_until | 2026-03-07T22:30:00+08:00 |
