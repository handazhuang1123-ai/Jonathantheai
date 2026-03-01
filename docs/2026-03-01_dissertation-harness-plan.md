---
title: 博士论文研究写作 — Harness 流水线方案
date: 2026-03-01
author: Claude Opus 4.6 (assisting zhuangba)
tags: [harness, dissertation, research, writing, pipeline]
status: draft
---

# 博士论文研究写作 — Harness 流水线方案

## 1. 核心思路

将 Harness 的 coding agent 架构（Initializer → Worker → MDIE 监控）泛化为**研究写作流水线**。核心洞察来自 Anthropic 的 [Effective Harnesses for Long-Running Agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)：

> 长时任务成功的关键不是模型多强，而是**结构化工作流能否跨会话传递上下文**。

博士论文天然适合这个模式：
- 可拆解为多阶段多子任务（→ issues.json）
- 每步有明确的输入和产出物（→ git commit）
- 有可检测的进度信号和质量标准（→ MDIE + Quality Check）
- 需要人类在关键节点介入（→ 壮爸/导师反馈环）

## 2. 参考架构与最佳实践

### 2.1 Agent Laboratory（EMNLP 2025）

[Agent Laboratory](https://github.com/SamuelSchmidgall/AgentLaboratory) 是目前最成熟的 LLM 研究助理框架，三阶段流程：

1. **Literature Review** — arXiv API 查询 → 摘要筛选 → 全文提取 → 精读整理
2. **Experimentation** — 代码生成 → 执行 → 评分 → 自我反思迭代
3. **Report Writing** — 结果 + 代码 + 文献 → 学术论文格式输出

关键设计：
- 角色分工（PhD Agent / Postdoc Agent / ML Engineer Agent）
- **Co-Pilot 模式**：每阶段完成后暂停，等待人类反馈再进入下一阶段
- 成本：gpt-4o 跑完整个流程 $2.33 / ~20 分钟（MiniMax M2.5 估算 ≤$1）
- 降低研究成本 84%（对比传统流程）

### 2.2 Anthropic Harness 模式

Anthropic 的双 agent 架构已在我们的 Harness 中验证：
- **Initializer Agent**：创建环境、特性列表、进度文件
- **Worker Agent**：逐特性实现，每完成一个 commit + 更新进度
- **Progress File**：跨会话上下文传递的核心（对应我们的 issues.json）

### 2.3 AI 辅助学术写作的共识（2026）

- Nature 调查：65%+ 学者认为 AI 辅助特定写作任务可接受
- 人类在环（Human-in-the-Loop）是学术界的底线要求
- **引用幻觉**是最大风险 — Deep Research / ChatGPT 的引用准确率极低
- 推荐工具链：Semantic Scholar API（文献发现）+ Zotero（引用管理）+ Scite.ai（引用验证）
- 系统性文献综述的 AI 辅助可节省 50-80% 时间，但仍需专家在每步把关

### 2.4 已知风险与对策

| 风险 | 严重性 | 对策 |
|------|--------|------|
| **引用幻觉**（编造不存在的论文） | 致命 | 每条引用必须通过 Semantic Scholar API 或 DOI 反查验证 |
| **论述浅薄**（缺乏深度分析） | 高 | 壮爸在每章完成后精读审查，标注"需深化"段落 |
| **学术不端嫌疑** | 高 | 所有 AI 辅助环节透明记录，遵循学校 AI 使用政策 |
| **上下文丢失**（跨会话遗忘） | 中 | progress.md + git history + issues.json 三重持久化 |
| **过度依赖 AI** | 中 | AI 只做"苦力"（搜索、整理、初稿），核心论证和创新由壮爸完成 |

## 3. 流水线设计

### 3.1 总体架构

```
壮爸 ──(Telegram)──→ Jonathan ──(tmux)──→ Research Harness
  ↑                      ↕                        ↓
  │                 MDIE 监控循环              产出物 (git)
  │                 质量检查 [Q]                   │
  │                      ↓                        ↓
  └──── 阶段性审查 ←── 汇报 ←──────────── progress.md
```

### 3.2 项目目录结构

```
~/projects/dissertation/
├── spec.md                     # 论文总体 spec（题目、RQ、方法论、章节框架）
├── progress.md                 # 跨会话进度文件（Harness 核心）
├── .issue_store/issues.json    # 当前阶段任务清单
│
├── literature/                 # Phase 1-2 产出
│   ├── search-log.md           # 搜索策略记录（关键词、数据库、日期）
│   ├── papers/                 # 文献卡片（每篇一个 YAML）
│   │   ├── smith2024-topic.yaml
│   │   └── ...
│   ├── synthesis-matrix.csv    # 综述矩阵
│   └── gaps.md                 # 研究空白分析
│
├── chapters/                   # Phase 3 产出
│   ├── ch1-introduction.md
│   ├── ch2-literature-review.md
│   ├── ch3-methodology.md
│   ├── ch4-results.md
│   └── ch5-discussion.md
│
├── review/                     # Phase 4 产出
│   ├── ch2-review-notes.md
│   └── ...
│
├── references.bib              # BibTeX 引文库
└── .harness/                   # Harness 配置
    ├── initializer_prompt.md
    ├── worker_prompt.md
    └── quality_checklist.md
```

### 3.3 阶段定义

#### Phase 1: 文献扫描（Literature Scanning）

**触发**：壮爸发送 "开始文献扫描" + 研究问题/关键词

**Initializer 任务**：
- 解析研究问题，拆分为搜索维度（每个维度 = 1 个 issue）
- 每个 issue 定义：关键词组合、目标数据库、筛选条件（年份/期刊/语言）
- 创建 `search-log.md` 记录搜索策略

**Worker 任务**（逐 issue 执行）：
- 通过 Semantic Scholar API / Google Scholar（Playwright）搜索
- 提取元数据：标题、作者、年份、摘要、引用数、DOI、PDF 链接
- 生成文献卡片 YAML：
  ```yaml
  id: smith2024-adaptive-learning
  title: "Adaptive Learning in Higher Education"
  authors: [Smith, J., Chen, L.]
  year: 2024
  source: Journal of Educational Technology
  doi: 10.1234/jet.2024.001
  citations: 47
  abstract: "..."
  relevance: high  # high/medium/low
  relevance_reason: "直接讨论了自适应学习的效果评估方法"
  tags: [adaptive-learning, assessment, higher-education]
  ```
- 每完成一个搜索维度 → git commit

**MDIE Monitor 检查**：
- 每个维度是否找到 ≥10 篇相关文献
- 去重率（跨维度重复论文占比）
- 高被引论文覆盖率（领域 top 引用是否在列）

**Quality Check**：
- 时间分布是否合理（不能全是老论文或全是新论文）
- 核心期刊覆盖率
- 搜索策略是否有明显盲区

**产出**：`literature/papers/` 下 N 篇文献卡片 + `search-log.md`

**壮爸审查点**：确认关键词覆盖、补充遗漏方向、决定哪些进入精读

---

#### Phase 2: 文献精读 & 综述矩阵（Deep Reading & Synthesis）

**触发**：壮爸确认 Phase 1 结果，标记哪些论文需要精读

**Initializer 任务**：
- 将标记的论文生成 issue（每篇 = 1 个 issue）
- 定义综述矩阵的列维度（壮爸指定或 AI 建议后确认）
  - 例：研究问题 | 方法论 | 样本量 | 核心发现 | 局限性 | 与本研究的关系

**Worker 任务**：
- 读取论文（PDF 全文或 abstract + introduction + conclusion）
- 按矩阵维度提取信息，填入卡片的扩展字段
- 识别论文间的关系（支持/矛盾/扩展）
- 每 5 篇完成后合并更新 `synthesis-matrix.csv`
- 精读完所有论文后生成 `gaps.md`（研究空白分析）

**MDIE Monitor 检查**：
- 矩阵填充完整性（是否每篇每列都有内容）
- 进度：已精读 / 总数

**Quality Check**：
- 抽样回查：随机选 3 篇，对比提取内容与原文是否一致
- 研究空白是否有充分证据支撑（不是凭空编造的 gap）

**产出**：`synthesis-matrix.csv` + `gaps.md` + 扩展后的文献卡片

**壮爸审查点**：确认综述矩阵、确认研究空白、与导师讨论后调整 RQ

---

#### Phase 3: 章节起草（Chapter Drafting）

**触发**：壮爸给出章节大纲 + 核心论点

**Initializer 任务**：
- 将章节拆分为 sections/subsections（每个 = 1 个 issue）
- 为每个 section 关联引用池（从 Phase 2 综述矩阵中筛选）
- 生成写作指令（学术风格、目标字数、引用密度要求）

**Worker 任务**：
- 按 section 顺序起草
- 嵌入引用标记 `[@smith2024]`，对应 `references.bib` 中的条目
- 保持 sections 间的逻辑衔接（每个 section 开头回顾上一 section 结论）
- 每完成一个 section → git commit

**MDIE Monitor 检查**：
- 各 section 字数是否达标
- 引用密度（每 500 字至少 N 个引用）
- section 间逻辑链是否连贯

**Quality Check（学术专项）**：
```
[Q] Academic Quality Check:
1. 引用验证 — 正文中 [@xxx] 是否都在 references.bib 中存在
2. 引用真实性 — 抽样 3 条引用，通过 DOI 反查确认论文真实存在
3. 论证链 — 每个 claim 是否有 evidence/citation 支撑
4. 术语一致性 — 关键术语全文是否统一
5. 格式规范 — 标题层级、图表编号连续性
6. 自我查重 — 与引用论文的文字重叠检查（避免无意抄袭）
```

**产出**：`chapters/chN-xxx.md` + 更新 `references.bib`

**壮爸审查点**：精读初稿、标注需深化/重写的段落、导师反馈后发起修订

---

#### Phase 4: 交叉审查 & 修订（Cross-Review & Revision）

**触发**：壮爸完成人工审查，或导师反馈回来

**Initializer 任务**：
- 将审查意见拆分为修订 issues（每条意见 = 1 个 issue）
- 分类：致命（逻辑矛盾）/ 重要（论述不足）/ 建议（语言润色）
- 优先级排序

**Worker 任务**：
- 按优先级逐条修订
- 修订后标注变更原因（对应哪条审查意见）
- 每条修订 → git commit（commit message 引用 issue 编号）

**MDIE Monitor 检查**：
- 致命/重要 issues 是否全部处理
- 修订是否引入了新问题

**Quality Check**：
- 修订后重跑 Phase 3 的学术质量检查
- 全文一致性（修改某章后其他章是否需要同步更新）

**产出**：修订后的章节文件 + `review/chN-review-notes.md`

**壮爸审查点**：确认修订质量，决定是否需要再一轮

## 4. 关键设计决策

### 4.1 Human-in-the-Loop 节点

```
Phase 1 完成 → [壮爸审查] → Phase 2
Phase 2 完成 → [壮爸审查 + 导师讨论] → Phase 3
Phase 3 每章完成 → [壮爸精读] → 修订指令
Phase 4 完成 → [壮爸 + 导师终审] → 下一章 or 提交
```

**设计原则**：AI 永远不自动进入下一 Phase。每个 Phase 完成后暂停，等壮爸显式触发。

### 4.2 引用完整性保障

引用幻觉是 AI 学术写作的最大风险。三层防线：

1. **源头控制**：Phase 1 的每篇文献都有 DOI / Semantic Scholar ID，不接受无法验证的来源
2. **写作约束**：Worker prompt 明确规定"只能引用 `papers/` 目录下存在的文献"
3. **自动验证**：Quality Check 抽样通过 API 反查 DOI，确认论文真实存在且内容匹配

### 4.3 跨会话上下文传递

```
progress.md（简要状态）
   + issues.json（任务清单）
   + git log（变更历史）
   + synthesis-matrix.csv（知识积累）
= 新 session 的 Worker 能在 30 秒内理解全局状态
```

### 4.4 与现有 Harness 的复用

| 现有 Harness 组件 | 论文版改动 |
|---|---|
| `initializer_prompt.md` | 从"分析代码需求"改为"分析研究任务，拆解搜索/阅读/写作子步骤" |
| `coding_prompt.md` → `worker_prompt.md` | 从"写代码"改为"执行研究任务（搜索/提取/写作）" |
| `MDIE.md` | Monitor 命令适配（不再只看 git log，还要检查文献卡片数、矩阵填充率、章节字数） |
| `APP_SPEC_GUIDE.md` → `research_spec_guide.md` | 从"app spec"改为"论文 spec"（RQ、方法论、章节框架） |
| `issues.json` | 结构不变，issue 内容从 feature 变为研究子任务 |
| `run_harness.sh` | 不变 |
| HEARTBEAT 触发 | 不变（30 分钟检查一次进度） |

**核心代码改动量极小** — 主要是换 prompt，架构和脚本基本复用。

## 5. 实施路线

### Step 1: 准备（~1 天）

- [ ] 确认论文领域和阶段（壮爸提供）
- [ ] 编写 `research_spec_guide.md`（论文 spec 模板）
- [ ] 改写 `initializer_prompt.md` 和 `worker_prompt.md`
- [ ] 适配 `MDIE.md` 的 Monitor 和 Quality Check

### Step 2: Phase 1 试跑（~2 天）

- [ ] 壮爸提供研究问题 + 初始关键词
- [ ] 跑一轮文献扫描，验证搜索 + 文献卡片生成 + 去重
- [ ] 壮爸审查产出质量，调整 prompt

### Step 3: Phase 2 试跑（~3 天）

- [ ] 壮爸标记精读论文
- [ ] 跑精读 + 综述矩阵生成
- [ ] 验证提取准确性（壮爸抽样回查）

### Step 4: Phase 3 试跑（~1 周）

- [ ] 壮爸给出一章大纲
- [ ] 跑章节起草
- [ ] 验证引用完整性、论证链、学术语言质量

### Step 5: 全流程打通

- [ ] 四阶段串联
- [ ] 导师反馈环集成
- [ ] 长期运行稳定性验证

## 6. 参考资料

- [Anthropic: Effective Harnesses for Long-Running Agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents) — 双 agent + progress file 的核心设计
- [Agent Laboratory (EMNLP 2025)](https://github.com/SamuelSchmidgall/AgentLaboratory) — 文献→实验→报告 三阶段框架，Co-Pilot 模式
- [LitLLM](https://litllm.github.io/) — RAG + LLM 重排序的文献综述生成
- [LLAssist](https://arxiv.org/html/2407.13993v3) — 开源文献筛选与相关性评估
- [AutoLit (Nested Knowledge)](https://pmc.ncbi.nlm.nih.gov/articles/PMC12552804/) — Human-in-the-Loop 系统性文献综述，验证节省 50-80% 时间
- [Cursor Long-Running Agents](https://cursor.com/blog/long-running-agents) — 52 小时自主运行的 agent 架构经验
- [A Vision for Auto Research with LLM Agents](https://arxiv.org/html/2504.18765v1) — 自动化研究全生命周期的愿景论文
- [Scite.ai](https://scite.ai/) — 引用关系分类（支持/反驳/提及），降低引用幻觉风险
- [Semantic Scholar API](https://api.semanticscholar.org/) — 免费学术论文搜索 API，支持引用网络探索
