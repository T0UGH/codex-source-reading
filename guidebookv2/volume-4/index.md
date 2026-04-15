---
title: 卷四｜控制面与 app-server 语义
status: draft
updated: 2026-04-14
---

# 卷四｜控制面与 app-server 语义

卷四回答：app-server 为什么不是另一套 runtime，而是建立在 core 之上的控制面 facade，以及这套控制面怎样站稳。

## 本卷读完后你应该得到什么

1. 知道 app-server 为什么不是平行 runtime，而是控制面 facade
2. 知道 listener task、协议投影、状态修正怎样共同撑起控制面
3. 知道 `ServerRequestResolved` 与 `DynamicToolCall` 在控制面语义里分别站在哪里
4. 知道 TUI 为什么越来越像跑在 app-server 之上，而不是直接抓 core

## 本卷五篇分别在纠正什么误判

- **01** app-server 不是另一套平行 runtime
- **02** 控制面不是“顺手包一层 UI”，而是靠 listener / 协议投影 / 状态修正一起站稳
- **03** `ServerRequestResolved` 不是“所有 request 最后都会经过的统一 resolved 事件”
- **04** `DynamicToolCall` 不是 resolved 模型的漏网之鱼，而是一次有意的语义分叉
- **05** TUI 越来越像跑在 app-server 之上，不是因为 core 不重要，而是因为正式控制面已经成立

## 目录

1. [为什么 app-server 不是另一套 runtime，而是建立在 core 之上的控制面 facade](./2026-04-12-Codex-卷四-01-为什么-app-server-不是另一套-runtime.md)
2. [listener task、协议投影与状态修正怎样把控制面撑起来](./2026-04-12-Codex-卷四-02-listener-task-协议投影与状态修正怎样把控制面撑起来.md)
3. [Codex 卷四 03｜`ServerRequestResolved` 到底覆盖了什么控制面语义](./2026-04-12-Codex-卷四-03-ServerRequestResolved-到底覆盖了什么控制面语义.md)
4. [Codex 卷四 04｜为什么 DynamicToolCall 不走 ServerRequestResolved](./2026-04-12-Codex-卷四-04-为什么-DynamicToolCall-不走-ServerRequestResolved.md)
5. [Codex 卷四 05｜为什么 TUI 越来越像跑在 app-server 之上，而不是直接抓 core](./2026-04-12-Codex-卷四-05-为什么-TUI-越来越像跑在-app-server-之上.md)

## 推荐阅读方式

- 如果你顺着全书读到这里：按 01 → 02 → 03 → 04 → 05 顺着读，先立 app-server 边界，再立控制面语义，最后再看 TUI 怎样站在这层之上
- 如果你最关心控制面 contract：先读 01、02、03
- 如果你最关心 TUI 与控制面的关系：先读 01、05

## 本卷最后要留下的一句话

> **app-server 不是另一套 runtime，而是建立在 core runtime 之上的统一控制面 facade。**

## 卷间导流

- **卷三** 解释这条工作线怎样持续存在
- **卷四** 解释这条 runtime 工作线怎样被暴露成稳定控制面
- **卷五** 会继续回答：动作怎样在这套系统里被落成执行会话