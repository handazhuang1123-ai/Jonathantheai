# 记忆文件格式规范

## 文件结构

项目根目录下的 `README.md` 是文件索引（阅读顺序 + TL;DR）。

四层架构：
- **索引层**: README.md（始终最新）
- **当前层**: `YYYY-MM-DD_topic.md`（status: current）— 按需加载，日常操作记忆
- **文档层**: `docs/`（深度分析报告、构建计划等参考文档）— 按需整篇加载，不纳入渐进式流程
- **归档层**: `archive/`（status: superseded）— 仅供追溯

## YAML Frontmatter

每个记忆文件必须包含：

```yaml
---
title: 描述性标题
date: YYYY-MM-DD
tags: [tag1, tag2]
depends_on: []
status: current  # current | superseded
last_updated: YYYY-MM-DD
---
```

## 容量限制

- 当前层 ≤8 个文件，单文件 ≤150 行
- archive 同主题 ≤3 个版本
- 归档操作：`mkdir -p archive && mv`，新文件加 `supersedes` 字段
