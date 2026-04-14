---
title: Codex 源码导读手册
date: 2026-04-14
---

# Codex 源码导读手册

> 一套按 **6 个认知台阶** 重组 OpenAI Codex 内部结构的中文源码导读手册。

这里是公开站点首页。当前站点直接以 **`codex-source-reading` 仓库内现有文章** 作为唯一内容来源，不再维护第二份正文副本。

## 从哪里开始

### 第一次进入这个项目
建议直接从这里开始：

- [00｜如何阅读这份导读](./00-guidebook/00-如何阅读这份导读.md)
- [01｜系统总图与分层](./00-guidebook/01-系统总图与分层.md)
- [02｜状态持久化与恢复](./00-guidebook/02-状态持久化与恢复.md)

### 如果你想先抓整套结构
可以直接按六卷主线进入：

1. [卷一｜系统入口与总图](./00-guidebook/00-如何阅读这份导读.md)
2. [卷二｜状态、持久化与恢复](./00-guidebook/02-状态持久化与恢复.md)
3. [卷三｜app-server、thread 与 turn 主线](./00-guidebook/03-app-server与thread-turn主线.md)
4. [卷四｜统一执行子系统](./00-guidebook/05-unified-exec执行子系统.md)
5. [卷五｜能力系统与高级子系统](./00-guidebook/06-capability与高级子系统.md)
6. [卷六｜附录、微缺口与继续深挖入口](./00-guidebook/11-关键函数索引.md)

## 这套手册在讲什么

这套手册不是带你“逛源码目录”，而是在回答一串更关键的问题：

- Codex 到底是什么系统
- 它为什么不是一个简单命令壳，而是一个有 runtime owner 的系统
- rollout、SQLite、thread、turn、listener、turn-history 这些对象为什么能让系统持续工作
- app-server 为什么更像控制面 facade，而不是另一套平行 runtime
- unified-exec 为什么不是普通执行器，而是一条 execution session 主线
- model transport、backend、review、guardian、realtime、collab、memories 又怎样把它推向更高层 runtime 组织

## 六卷各自回答什么

- **卷一**：先看清 Codex 到底是什么系统，以及该怎么读这套材料
- **卷二**：看状态、持久化和恢复机制为什么能让系统续跑
- **卷三**：看 app-server、thread、turn、listener、turn-history 怎样形成控制面主线
- **卷四**：看执行能力怎样被统一收口成 execution session
- **卷五**：看能力系统、模型传输、审查基础设施和高级 runtime 子系统
- **卷六**：看索引、开放问题和微缺口怎样支撑继续深挖

## 推荐阅读路线

### 路线 A：第一次系统阅读
- [00｜如何阅读这份导读](./00-guidebook/00-如何阅读这份导读.md)
- [01｜系统总图与分层](./00-guidebook/01-系统总图与分层.md)
- [02｜状态持久化与恢复](./00-guidebook/02-状态持久化与恢复.md)
- 然后按 03 → 15 顺序继续

### 路线 B：只想抓 runtime 主线
- [02｜状态持久化与恢复](./00-guidebook/02-状态持久化与恢复.md)
- [03｜app-server 与 thread-turn 主线](./00-guidebook/03-app-server与thread-turn主线.md)
- [04｜turn-history 语义层](./00-guidebook/04-turn-history语义层.md)
- [05｜unified-exec 执行子系统](./00-guidebook/05-unified-exec执行子系统.md)

### 路线 C：只想抓高级系统与 future-looking 问题
- [06｜capability 与高级子系统](./00-guidebook/06-capability与高级子系统.md)
- [09｜review 工作流与 guardian 审查基础设施](./00-guidebook/09-review工作流与guardian审查基础设施.md)
- [10｜realtime、collab 与 memory 迁移专题](./00-guidebook/10-realtime-collab与memory迁移专题.md)
- [13｜open questions 与后续深挖方向](./00-guidebook/13-open-questions与后续深挖方向.md)

## 当前入口说明

- GitHub Pages 直接读取这个仓库里的现有文章，不再维护第二套站点正文
- `00-guidebook/` 是主阅读层
- `01-source-notes/` 是证据层
- `00-index/`、`02-call-chain-drafts/`、`03-boundary-judgments/` 提供继续深挖入口
