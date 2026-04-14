---
title: Codex 源码导读手册
date: 2026-04-14
---

# Codex 源码导读手册

> 一套按 **6 个认知台阶** 重组 OpenAI Codex 内部结构的中文源码导读手册。

当前站点主入口已经切到 **guidebook v2**。其中：

- 卷一沿用仓库中已经稳定的入口与系统总图
- 从卷二开始，全部切换到最近这一轮新成稿内容
- 老的 `00-guidebook/02-15` 已移入 `trash/2026-04-14-guidebookv2-replaced-content/`

## 从哪里开始

### 第一次进入这个项目
建议直接从这里开始：

- [开始阅读｜Guidebook v2 总览](./guidebookv2/README.md)
- [卷一｜系统入口与总图](./guidebookv2/volume-1/index.md)
- [卷二｜runtime core 主线](./guidebookv2/volume-2/index.md)

### 如果你想先抓整套结构
可以直接按六卷导读进入：

1. [卷一｜系统入口与总图](./guidebookv2/volume-1/index.md)
2. [卷二｜runtime core 主线](./guidebookv2/volume-2/index.md)
3. [卷三｜状态、恢复与持续工作线](./guidebookv2/volume-3/index.md)
4. [卷四｜控制面与 app-server 语义](./guidebookv2/volume-4/index.md)
5. [卷五｜统一执行子系统](./guidebookv2/volume-5/index.md)
6. [卷六｜审查协作与高级 runtime](./guidebookv2/volume-6/index.md)

## 这套手册在讲什么

这套手册不是带你“逛源码目录”，而是在回答一串更关键的问题：

- Codex 到底是什么系统
- 一条请求怎么真正进入 runtime core 主工作线
- rollout、replay、thread、turn、pending request 怎样让系统持续工作
- app-server 为什么更像控制面 facade，而不是另一套平行 runtime
- unified-exec 怎样把执行动作装成正式 execution session
- `/review`、guardian、realtime、collab、AgentControl、memories 又怎样把系统推向更高层 runtime 组织

## 当前入口说明

- `guidebookv2/` 是当前正式阅读入口
- `00-guidebook/00-01` 继续承担卷一入口角色
- 证据层与边界判断仍保留在仓库中，可通过导航的“证据层入口”继续下钻
