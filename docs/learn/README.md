# Hermes Learn

这个目录用于沉淀 Hermes Agent 的架构学习笔记。内容面向代码阅读和二次开发，重点解释系统边界、运行链路和关键概念，而不是用户使用教程。

## 文档列表

- [系统架构分析](architecture.md)：从整体分层、核心模块、工具系统、插件体系、会话持久化等角度理解 Hermes。
- [Profile 机制](profile.md)：解释 Hermes profile 如何通过 `HERMES_HOME` 隔离配置、密钥、会话、技能、记忆和日志。
- [一次对话的数据链路](conversation-data-flow.md)：以 TUI 中的一次用户提问为例，串起入口、Agent、模型、工具、持久化和回传。

## 推荐阅读顺序

1. 先读 [系统架构分析](architecture.md)，建立整体地图。
2. 再读 [Profile 机制](profile.md)，理解为什么同一套代码会有多套独立运行状态。
3. 最后读 [一次对话的数据链路](conversation-data-flow.md)，把前两篇概念落到一轮真实执行过程里。
