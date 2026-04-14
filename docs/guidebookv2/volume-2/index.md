---
title: 卷二｜状态、持久化与恢复
status: draft
updated: 2026-04-14
---

# 卷二｜状态、持久化与恢复

卷二回答：**Codex 为什么不是“一次命令执行就结束”，而是一个可以恢复、续跑和维持状态的系统。**

## 目录

1. [卷二 01｜状态持久化与恢复](./01-state-persistence-and-recovery.md)

## 这一卷要留下什么

- rollout 是正文真相源
- SQLite state DB 更像元数据 / 索引 / sidecar 层
- 恢复更像 replay，而不是 ORM 式对象重建
