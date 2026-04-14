---
title: 卷一｜Codex 系统全景导论
status: draft
updated: 2026-04-14
---

# 卷一｜Codex 系统全景导论

卷一先回答一个总问题：**Codex 到底是什么系统，应该从哪里进入它的整体结构。**

这一卷对应公开站点的起步入口，先把“怎么读”和“系统总图”钉住，再继续进入后面的 runtime 主线。

## 目录

1. [卷一 01｜如何阅读这份 Codex 导读](./01-how-to-read-this-guidebook.md)
2. [卷一 02｜系统总图与分层](./02-system-overview-and-layering.md)

## 这一卷要留下什么

- `codex-cli/` 是分发壳，不是主体
- `codex-rs/cli` 是统一入口层
- `core` 与 `ThreadManager` 才更接近 runtime owner
- TUI / app-server / mcp-server 都应按“产品面 / 控制面 / 暴露面”去看
