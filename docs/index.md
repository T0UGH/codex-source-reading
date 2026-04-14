---
title: Codex 源码导读手册
date: 2026-04-14
---

# Codex 源码导读手册

> 一套按 **6 个认知台阶** 重组 OpenAI Codex 内部结构的中文源码导读手册。

这里是公开站点首页。当前默认阅读入口已经切到 **guidebook v2**。

## 从哪里开始

### 第一次进入这个项目
建议直接从这里开始：

- [开始阅读｜Guidebook v2 总览](./guidebookv2/README.md)
- [卷一｜Codex 系统全景导论](./guidebookv2/volume-1/index.md)
- [卷二｜状态、持久化与恢复](./guidebookv2/volume-2/index.md)

### 如果你想先抓整套结构
可以直接按六卷导读进入：

1. [卷一｜Codex 系统全景导论](./guidebookv2/volume-1/index.md)
2. [卷二｜状态、持久化与恢复](./guidebookv2/volume-2/index.md)
3. [卷三｜app-server、thread 与 turn 主线](./guidebookv2/volume-3/index.md)
4. [卷四｜统一执行子系统](./guidebookv2/volume-4/index.md)
5. [卷五｜能力系统与高级子系统](./guidebookv2/volume-5/index.md)
6. [卷六｜附录、微缺口与继续深挖入口](./guidebookv2/volume-6/index.md)

## 这套手册在讲什么

这套手册不是带你“逛源码目录”，而是在回答一串更关键的问题：

- Codex 到底是什么系统
- 它为什么不是一个简单命令壳，而是一个有 runtime owner 的系统
- rollout、SQLite、thread、turn、listener、turn-history 这些对象为什么能让系统持续工作
- app-server 为什么更像控制面 facade，而不是另一套平行 runtime
- unified-exec 为什么不是普通执行器，而是一条 execution session 主线
- model transport、backend、review、guardian、realtime、collab、memories 又怎样把它推向更高层 runtime 组织

## 六卷各自回答什么

- **卷一**：先看清 Codex 到底是什么系统
- **卷二**：看状态、持久化和恢复机制为什么能让系统续跑
- **卷三**：看 app-server、thread、turn、listener、turn-history 怎样形成控制面主线
- **卷四**：看执行能力怎样被统一收口成 execution session
- **卷五**：看能力系统、模型传输、审查基础设施和高级 runtime 子系统
- **卷六**：看索引、开放问题和微缺口怎样支撑继续深挖

## 推荐阅读路线

### 路线 A：第一次系统阅读
- [Guidebook v2 总览](./guidebookv2/README.md)
- [卷一](./guidebookv2/volume-1/index.md)
- [卷二](./guidebookv2/volume-2/index.md)
- 然后按卷三到卷六顺序继续

### 路线 B：只想抓 runtime 主线
- [卷二｜状态、持久化与恢复](./guidebookv2/volume-2/index.md)
- [卷三｜app-server、thread 与 turn 主线](./guidebookv2/volume-3/index.md)
- [卷四｜统一执行子系统](./guidebookv2/volume-4/index.md)

### 路线 C：只想抓高级系统与 future-looking 问题
- [卷五｜能力系统与高级子系统](./guidebookv2/volume-5/index.md)
- [卷六｜附录、微缺口与继续深挖入口](./guidebookv2/volume-6/index.md)

## 当前入口说明

- `guidebookv2/` 是当前正式阅读入口
- 仓库根目录下的 `00-guidebook/`、`01-source-notes/` 等旧资产仍保留在仓库中，但不作为公开站点主导航
- 站点导航默认按 guidebook v2 的六卷结构展开
