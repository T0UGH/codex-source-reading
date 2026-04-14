---
title: 卷三｜app-server、thread 与 turn 主线
status: draft
updated: 2026-04-14
---

# 卷三｜app-server、thread 与 turn 主线

卷三回答：**core runtime 的 live event、thread 状态和 turn 语义，怎样被 app-server 持续投影成一个可连接、可恢复、可 replay 的控制面。**

## 目录

1. [卷三 01｜app-server 与 thread-turn 主线](./01-app-server-and-thread-turn-mainline.md)
2. [卷三 02｜turn-history 语义层](./02-turn-history-semantics.md)

## 这一卷不提前展开什么

- 不把这一卷写成单纯协议枚举表
- 不把这一卷提前写成 unified-exec 实现卷
- 不把 listener / projection / reconciliation 的边界混成一个大状态机
