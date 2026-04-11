---
title: resolve_thread_status(...) 与 track_current_turn_event(...) 是一对竞态修正组合
date: 2026-04-11
tags:
  - Codex
  - 源码共读
  - thread-status
  - turn-state
  - race
source_files:
  - /Users/wangguiping/workspace/codex/codex-rs/app-server/src/thread_status.rs
  - /Users/wangguiping/workspace/codex/codex-rs/app-server/src/thread_state.rs
  - /Users/wangguiping/workspace/codex/codex-rs/app-server/src/codex_message_processor.rs
status: done
---

# Codex 源码共读 48：resolve_thread_status(...) 与 track_current_turn_event(...) 是一对竞态修正组合

## 这篇看什么

前面已经分别拆过：
- `set_thread_status_and_interrupt_stale_turns(...)`
- `populate_thread_turns(...)`
- `handle_turn_complete(...)`

但再往下会发现，有两块很容易单看都觉得“很小”的代码：
- `resolve_thread_status(...)`
- `track_current_turn_event(...)`

单独看它们都像小 helper，
但放在一起，其实是一组很关键的组合。

这篇就把它们放在一起讲。

## 先给主结论

如果先留一句话，我会写：

> `track_current_turn_event(...)` 和 `resolve_thread_status(...)` 并不是两个独立小 helper，而是一组配套的竞态修正机制：前者在 listener 收到原始 event 时，优先把当前 turn 的本地事实（例如 started_at、active turn history）写进 `ThreadState`；后者则在需要向客户端返回 thread status 时，用这些本地事实去修补 watch-manager 可能滞后的 `Idle/NotLoaded` 判断，从而把“事件已经到了，但 watch-state 还没跟上”的短窗口尽量抹平。

再压缩一点：

> **前者积累 live 证据，后者拿这些证据修正滞后的 watch 状态。**

---

## 第一层：`track_current_turn_event(...)` 负责建立本地 turn 事实

这个函数虽然很小，但做了两件关键事：

1. 如果是 `TurnStarted`，先把 `started_at` 写进 `turn_summary`
2. 再把 event 喂进 `current_turn_history.handle_event(...)`

这就意味着：
- 只要 listener 已经看到 event
- thread-local turn state 就会先行更新

这个更新发生在：
- 任何 bespoke protocol emission 之前

所以它建立的是：

> **listener 视角下的局部事实真相。**

这比 watch manager 更靠近事件源。

---

## 第二层：`resolve_thread_status(...)` 负责承认 watch manager 可能落后

它的逻辑非常克制：
- 只有 `status == Idle || NotLoaded`
- 且 `has_in_progress_turn == true`
- 才会强制升成 `Active`

说明作者对它的定位很明确：

- 不是全能状态机
- 只是一个小范围的 correction shim

也就是说，它不是在“重新计算状态”，
而是在做：

> **针对已知 race window 的最小必要修补。**

---

## 第三层：为什么这两者必须一起看

单看 `resolve_thread_status(...)` 会问：
- 这个 `has_in_progress_turn` 从哪来？

单看 `track_current_turn_event(...)` 会问：
- 它更新了 active turn，又怎么样？

把它们连起来就清楚了：

1. listener 先通过 `track_current_turn_event(...)` 积累本地 active-turn 事实
2. later response/read/resume 路径再根据这些事实得到：
   - `has_live_in_progress_turn`
3. `resolve_thread_status(...)` 再用它修补 thread status

所以这两者本来就是一条链上的上下游。

---

## 第四层：这对组合揭示了 app-server 内部其实有两套“真相源”

### 真相源 A：watch manager
提供的是：
- loaded / running / waiting / system error 等归纳态

### 真相源 B：listener-local turn state
提供的是：
- 当前 active turn 是否存在
- started_at
- active turn snapshot

这两者都是真相，但时间颗粒度不同：
- watch manager 偏总结态
- listener-local state 偏事件近场态

所以 `resolve_thread_status(...)` + `track_current_turn_event(...)` 组合，
本质上是在解决：

> **不同时间颗粒度真相源的短时不一致。**

这个判断我觉得特别重要。

---

## 第五层：`track_current_turn_event(...)` 不只是维护 active turn，也在决定何时清空“当前 turn 缓存”

除了写 started_at，它还会：
- 在 `TurnAborted` / `TurnComplete` 且没有 active turn 时 reset 当前 turn history

这说明它不只是 accumulator，
也是当前 turn 局部缓存的清理边界之一。

所以这个函数更准确地说是：

> **current-turn state updater + resetter**

而不只是“记录一下 started_at”。

---

## 第六层：`resolve_thread_status(...)` 为什么不覆盖 `SystemError`

这个细节很值。

它只会修：
- `Idle`
- `NotLoaded`

不会把：
- `SystemError`

改成 `Active`。

这说明在状态优先级里：

> **明确错误 > 推测活跃。**

也就是说，live turn 证据只能修补“还没跟上”的状态，
不能覆盖“已经明确错误”的状态。

这是一个很成熟的保守选择。

---

## 第七层：从设计上看，这是典型的“近场状态修补远场投影”

我会把它总结成：

- `track_current_turn_event(...)` = 近场状态积累
- `resolve_thread_status(...)` = 远场投影修补

这和很多实时系统的设计很像：
- 最近发生的事实，先在局部缓存里最先可见
- 更外层汇总状态稍后才更新
- 对外读时用局部事实补一刀

Codex 在这里做得其实很漂亮。

---

## 一句角色定义

如果按源码共读风格压一句话，我会这么写：

> **`track_current_turn_event(...)` 和 `resolve_thread_status(...)` 是 app-server 里把“live turn 近场事实”转译成“对客户端更稳定 thread status”的竞态修正组合。**

---

## 为什么这层设计是对的

### 1. 不指望 watch manager 永远先于事件到达
现实里做不到。

### 2. 用最小修补而不是重算整套状态
风险更低。

### 3. 让 listener-local state 真正发挥价值
否则 active turn snapshot 只会停留在本地缓存，不能改善对外表现。

---

## 继续细拆的话，下一篇最自然的点

最自然继续拆的是：

1. `ThreadHistoryBuilder::active_turn_snapshot(...)`
2. `loaded_thread_status(...)`
3. `note_turn_started(...)`
