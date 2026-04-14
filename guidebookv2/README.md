---
title: Codex 源码导读手册 v2
date: 2026-04-14
status: draft
---

# Codex 源码导读手册 v2

> 当前公开入口已经切到 **guidebook v2**。这版不再沿用旧 `00-guidebook/02-15` 的主干顺序，而是按 6 卷认知递进来组织整本书：先建立整机总图，再进入 runtime core、持续性、控制面、执行子系统与高级 runtime。

## 先读这一句

> **这 6 卷不是材料分桶，而是一条连续问题链：先看清 Codex 到底是什么系统，再看它怎样跑起来、怎样持续存在、怎样把 runtime 暴露成控制面、怎样把动作落成可管理执行、最后怎样长出审查、协作与更高层 runtime 组织。**

## 当前卷结构

1. [卷一｜系统入口与总图](./volume-1/index.md)
2. [卷二｜runtime core 主线](./volume-2/index.md)
3. [卷三｜状态、恢复与持续工作线](./volume-3/index.md)
4. [卷四｜控制面与 app-server 语义](./volume-4/index.md)
5. [卷五｜统一执行子系统](./volume-5/index.md)
6. [卷六｜审查协作与高级 runtime](./volume-6/index.md)

## 这版入口现在意味着什么

- 卷一已经不再只是旧文占位，而开始承担 **整机 preview / 系统前门** 的职责
- 卷二到卷六分别沿着卷一先立起的系统图，逐卷补 runtime 主体、持续性、控制面、执行链与高级 runtime
- 旧的 `00-guidebook/02-15` 已移动到 `trash/2026-04-14-guidebookv2-replaced-content/` 保留，不再作为主阅读入口

## 推荐阅读方式

- **第一次进入**：从 [卷一](./volume-1/index.md) 开始，按卷一 → 卷二 → 卷三顺着读
- **只想先抓 runtime 主线**：卷一读完后直接进 [卷二](./volume-2/index.md)
- **只想看控制面**：卷一读完后跳 [卷四](./volume-4/index.md)
- **只想看执行子系统**：卷一读完后跳 [卷五](./volume-5/index.md)

## 当前对卷一的期待

卷一读完后，读者至少应该能回答：

1. Codex 到底是什么系统，不是什么系统？
2. CLI、TUI、app-server、core 分别处在哪一层？
3. `ThreadManager`、`CodexThread`、`Codex` 大致分别扮演什么角色？
4. 一条最基础的 interactive 请求大概怎么跑？
5. 后面的卷二到卷六分别是在补哪一块认识？
