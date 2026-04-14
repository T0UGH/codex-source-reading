---
title: 卷一｜系统入口与总图
status: draft
updated: 2026-04-14
---

# 卷一｜系统入口与总图

卷一不负责先把某条机制讲透，而是先回答一个更前面的问题：**Codex 到底是什么系统，它由哪些主要组件构成，它们分别处在哪一层，后面的卷二到卷六又分别是在补哪一块认识。**

如果这一步不先立住，后面读 runtime core、状态恢复、app-server、unified-exec、review / guardian / collab，就会一直像在看局部零件，而不是在沿着同一张系统图往里走。

## 本卷读完后你应该得到什么

1. 知道 Codex 不是 npm CLI、不是单纯 TUI、也不是一坨很厚的 app-server
2. 对 CLI、TUI、app-server、core、ThreadManager、CodexThread、Codex、recovery、unified-exec、skills / MCP、review / guardian / collab / memories 建立第一次正确认识
3. 能口头讲出一条最基础的 interactive 主请求流大概怎么跑
4. 知道卷二到卷六分别是在补哪一块认识

## 目录

1. [Codex 卷一 01：Codex 到底是什么系统：从壳到主体的第一张总图](./2026-04-14-Codex-卷一-01-Codex-到底是什么系统-从壳到主体的第一张总图-v1.md)
2. [Codex 卷一 02：一条最基础的 interactive 主请求流怎么跑](./2026-04-14-Codex-卷一-02-一条最基础的-interactive-主请求流怎么跑-v1.md)
3. [Codex 卷一 03：Codex 各组件第一次认识与六卷地图](./2026-04-14-Codex-卷一-03-Codex-各组件第一次认识与六卷地图-v1.md)

## 推荐阅读方式

- 第一次进入这本书：从本卷 01 开始，按 01 → 02 → 03 顺着读
- 如果你已经知道 Codex 大致分层，只想立 runtime 主线：读完本卷 01 后直接进 [卷二](../volume-2/index.md)
- 如果你最关心控制面和 app-server：本卷 01、02 读完后跳 [卷四](../volume-4/index.md)
- 如果你最关心执行链：本卷 01、02 读完后跳 [卷五](../volume-5/index.md)

## 卷间导流

- **卷二**：把卷一里 preview 过的 runtime core 主工作回合真正展开
- **卷三**：解释这条工作线为什么能持续、能恢复、能继续
- **卷四**：解释 app-server 怎样把 runtime 暴露成稳定控制面
- **卷五**：解释 unified-exec 怎样把动作装成可管理执行会话
- **卷六**：解释 review / guardian / collab / memories 为什么不是边角功能，而是更高层 runtime 组织的一部分

## 辅助文件

- [卷一写作卡片](./2026-04-14-Codex-卷一系统入口与总图-README-writing-cards-v1.md)
