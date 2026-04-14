---
title: 卷三｜状态、恢复与持续工作线
status: draft
updated: 2026-04-14
---

# 卷三｜状态、恢复与持续工作线

卷三回答：Codex 为什么不是跑完就散，而是能靠 rollout、replay、thread、turn 和 pending request 把工作线重新接起来。

## 目录

1. [为什么 rollout 才是正文真相源，而 SQLite 不是恢复核心](./2026-04-12-Codex-卷三-01-为什么-rollout-才是正文真相源而-SQLite-不是恢复核心-v1.md)
2. [为什么恢复更像 replay，而不是“查库拼对象”](./2026-04-12-Codex-卷三-02-为什么恢复更像-replay-而不是-查库拼对象-v1.md)
3. [thread、turn 与 pending request 是怎么组成持续工作线的](./2026-04-12-Codex-卷三-03-thread-turn-与-pending-request-是怎么组成持续工作线的-v1.md)
4. [为什么 turn-history 不是 event log 镜像，而是一层 semantic projection](./2026-04-12-Codex-卷三-04-为什么-turn-history-不是-event-log-镜像-v1.md)
5. [active_turn_snapshot、handle_user_message 与当前工作视图的边界](./2026-04-12-Codex-卷三-05-active-turn-snapshot-handle-user-message-与当前工作视图的边界-v1.md)

## 辅助文件

本卷对应的写作卡片与执行说明也已一并迁入仓库，但不放进主导航。
