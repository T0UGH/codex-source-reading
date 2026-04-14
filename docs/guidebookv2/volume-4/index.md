---
title: 卷四｜统一执行子系统
status: draft
updated: 2026-04-14
---

# 卷四｜统一执行子系统

卷四回答：**Codex 怎么把执行动作装成真正的 execution session，而不是只跑一条命令。**

## 目录

1. [卷四 01｜unified-exec 执行子系统](./01-unified-exec-runtime.md)

## 这一卷要留下什么

- `exec.rs` 是 primitive layer
- `unified_exec` 是 sessionful agent-facing execution subsystem
- transcript / delta / process-store / end-event 要放在一条闭环里看
