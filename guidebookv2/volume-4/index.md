---
title: 卷四｜控制面与 app-server 语义
status: draft
updated: 2026-04-14
---

# 卷四｜控制面与 app-server 语义

卷四回答：app-server 为什么不是另一套 runtime，而是建立在 core 之上的控制面 facade，以及这套控制面怎样站稳。

## 目录

1. [为什么 app-server 不是另一套 runtime，而是建立在 core 之上的控制面 facade](./2026-04-12-Codex-卷四-01-为什么-app-server-不是另一套-runtime-v1.md)
2. [listener task、协议投影与状态修正怎样把控制面撑起来](./2026-04-12-Codex-卷四-02-listener-task-协议投影与状态修正怎样把控制面撑起来-v1.md)
3. [Codex 卷四 03｜`ServerRequestResolved` 到底覆盖了什么控制面语义](./2026-04-12-Codex-卷四-03-ServerRequestResolved-到底覆盖了什么控制面语义-v1.md)
4. [Codex 卷四 04｜为什么 `DynamicToolCall` 不走 `ServerRequestResolved`](./2026-04-12-Codex-卷四-04-为什么-DynamicToolCall-不走-ServerRequestResolved-v1.md)
5. [Codex 卷四 05｜为什么 TUI 越来越像跑在 app-server 之上，而不是直接抓 core](./2026-04-12-Codex-卷四-05-为什么-TUI-越来越像跑在-app-server-之上-v1.md)

## 辅助文件

本卷对应的写作卡片与执行说明也已一并迁入仓库，但不放进主导航。
