# OpenClaw Jonathantheai - 项目指令

本目录记录 **Jonathan**（OpenClaw AI agent）的运行、调试与优化过程。

## 你是谁

你在协助壮爸管理 Jonathan — 一个运行在 Linux 服务器上的 OpenClaw AI agent。

## 可用命令

- `/jonathan-context` — 渐进式加载 Jonathan 背景资料（新 session 首先运行）
- `/jonathan-update-memory` — 操作结束后更新记忆文件

## 文件结构

项目根目录下的 `README.md` 是文件索引（阅读顺序 + TL;DR）。

三层架构：
- **索引层**: README.md（始终最新）
- **当前层**: `YYYY-MM-DD_topic.md`（status: current）— 按需加载
- **归档层**: `archive/`（status: superseded）— 仅供追溯

## 规则

- 永远用中文
- 不要一次性读取所有文件，按 README 中的优先级和任务需要渐进加载
- frontmatter 中 `status` 含 `SENSITIVE` 或 tags 含 `sensitive` 的文件，仅在明确需要时读取
- 修改记忆文件时遵循 `/jonathan-update-memory` 中的规范
- 如果用户直接提出任务而没有先运行 `/jonathan-context`，你应主动先读取 README.md 获取 TL;DR 和阅读顺序，再根据任务需要加载相关文件，然后再开始工作
