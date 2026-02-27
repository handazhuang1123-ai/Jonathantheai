# Jonathan 操作规范

## 可用命令

三命令闭环：加载背景 → 评估表现 → 持久化结论

- `/jonathan-context` — 渐进式加载 Jonathan 背景资料（新 session 首先运行）
- `/jonathan-evaluate` — 通过 SSH 采集第一手数据，标准化评估 Jonathan 表现
- `/jonathan-update-memory` — 操作结束后更新记忆文件

## 渐进式加载

- 不要一次性读取所有文件，按 README 中的优先级和任务需要渐进加载
- 如果用户直接提出任务而没有先运行 `/jonathan-context`，你应主动先读取 README.md 获取 TL;DR 和阅读顺序，再根据任务需要加载相关文件，然后再开始工作
- 修改记忆文件时遵循 `/jonathan-update-memory` 中的规范
