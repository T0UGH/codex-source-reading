---
title: active_turn_snapshot(...) 是当前 turn 的对外投影口
date: 2026-04-11
tags:
  - Codex
  - 源码共读
  - active-turn
  - snapshot
  - projection
source_files:
  - /Users/wangguiping/workspace/codex/codex-rs/app-server-protocol/src/protocol/thread_history.rs
  - /Users/wangguiping/workspace/codex/codex-rs/app-server/src/thread_state.rs
status: done
---

# Codex 源码共读 51：active_turn_snapshot(...) 是当前 turn 的对外投影口

## 这篇看什么

`active_turn_snapshot(...)` 看起来特别小：
- 在 `ThreadState` 里只是转发
- 在 `ThreadHistoryBuilder` 里也就一行链式调用

但它其实很关键，因为 app-server 对“当前 turn 是否存在、长什么样”这件事，最终就是靠它向外暴露。

## 先给主结论

如果先留一句话，我会写：

> `active_turn_snapshot(...)` 的本质不是简单 getter，而是 current-turn reducer 对外暴露的稳定投影口：有 active pending turn 时，它把尚未 finalize 的 `PendingTurn` 临时 materialize 成一个 `Turn` 快照；没有 active pending turn 时，则退化为返回最后一个已完成 turn。这样做让 resume/rejoin/status-repair 路径都能在不打破 reducer 内部状态机的前提下，读取“当前最接近真实”的 turn 视图。

再压缩一点：

> **它不是读内部状态，而是给外部一份可消费的当前 turn 视图。**

---

## 第一层：它不是直接暴露 `PendingTurn`，而是投影成 `Turn`

实现很直接：

- 如果 `current_turn` 存在：`Turn::from(pending)`
- 否则：`self.turns.last().cloned()`

这个小设计很成熟。

因为它避免了两件事：
1. 外部知道 `PendingTurn` 内部结构
2. 外部需要理解“active turn 尚未 finalize”这一层 builder 语义

换句话说：

> **内部是 reducer state，外部拿到的是协议层 `Turn` 视图。**

这就是标准的 projection 边界。

---

## 第二层：有 active turn 时，它返回的是“未封口但可读”的近场快照

只要 `current_turn` 还在，`active_turn_snapshot(...)` 就优先把它 materialize 成 `Turn`。

这意味着它返回的不是：
- 只包含 finalized history 的保守视图

而是：
- **把当前正在进行中的 turn 也显式暴露出来**

这和前面 `track_current_turn_event(...)` / `resolve_thread_status(...)` 那组逻辑刚好接上：
- listener 一边更新 current-turn reducer
- 对外读取时一边通过 snapshot 把这份近场事实拿出来

所以它是 live 事实外露的关键接口。

---

## 第三层：没有 active turn 时退化为 last completed turn，是个很实用的兼容选择

这里很多人会下意识觉得：
- 没 active turn 就该 `None`

但 Codex 没这么做。

它会回落到：
- `self.turns.last().cloned()`

这说明作者要的不是“严格意义上的 active-only snapshot”，
而是：

> **当前 thread history 里最值得给上层消费的那个 turn 视图。**

这对 resume/rejoin/UI 很实用，
因为很多时候客户端真正想知道的是：
- 最近这个 turn 现在是什么样

而不一定非要区分它是不是 pending。

当然，这个命名会让人有点误读空间。
所以如果写 guidebook，我会专门提醒：

> `active_turn_snapshot(...)` 语义更接近 `current_or_last_turn_snapshot(...)`。

---

## 第四层：它依赖 `Turn::from(PendingTurn)`，所以 snapshot 不是旁路拼装，而是复用 finalize 语义

这点很值。

当前 turn 的快照不是另起一套 struct manually 拼出来，
而是直接走：
- `Turn::from(&PendingTurn)`

这意味着：
- pending snapshot 和 finalized turn 使用同一套 shape
- 状态字段、items、error、时间字段的呈现方式统一

所以这不是为了偷懒，
而是明确在追求：

> **当前 turn 视图与最终 turn 视图尽量同构。**

这能极大减少客户端分支判断。

---

## 第五层：它和 `has_active_turn()` 要一起看，不能单独理解

单看 `active_turn_snapshot(...)` 很容易误判成：
- snapshot 存在 == 当前有 active turn

这是不对的。

因为它在没有 `current_turn` 时还会返回最后一个 finalized turn。

所以正确的理解是：
- `active_turn_snapshot()` 回答“当前最可消费的 turn 视图是什么”
- `has_active_turn()` 回答“现在是否真的还有 pending active turn”

这两个信号是配套的。

这也是为什么 `thread_state.track_current_turn_event(...)` 在 `TurnComplete/Aborted` 后，如果 `has_active_turn()` 为 false，会直接 reset reducer，防止局部缓存长期挂着。

---

## 第六层：测试已经明确说明它被当成 live in-progress turn 的读取口

thread_history 里的测试不是拿它验证“最近最后一个 turn”，
而是拿它验证：
- `TurnStarted + UserMessage + ApplyPatch...`
- 之后 snapshot 仍然是 `InProgress`
- items 已经包含进行中的 patch/file-change

这说明它在作者心里的主要用途就是：

> **读取当前 in-progress turn 的可展示快照。**

只不过为了兼容/易用，它才多给了 last-turn fallback。

---

## 第七层：`ThreadState.active_turn_snapshot(...)` 只是转发，说明真正的语义权威在 builder

`ThreadState` 里这个函数完全是：
- `self.current_turn_history.active_turn_snapshot()`

这表明 app-server 没有另做一套 snapshot 语义。

所以权威点还是：
- `ThreadHistoryBuilder`

这和前面的判断一致：
- thread state 不重新解释 turn 语义
- 它只是承载、暴露 builder 的结果

---

## 一句角色定义

如果按源码共读风格压一句话，我会这么写：

> **`active_turn_snapshot(...)` 是 current-turn reducer 向 app-server 外部暴露“当前最可消费 turn 视图”的投影口：优先给未封口 active turn，缺失时再回退到最后一个已知 turn。**

---

## 为什么这层设计是对的

### 1. 不暴露 `PendingTurn` 内部结构
边界清楚。

### 2. 当前 turn 与最终 turn 共用同构视图
客户端更省事。

### 3. live/rejoin/resume 都能读到近场事实
改善交互真实感。

### 4. fallback 到 last turn
兼容性和可用性都更好。

---

## 继续细拆的话，下一篇最自然的点

最自然继续拆的是：

1. `has_active_turn(...)`
2. `handle_user_message(...)`
3. `populate_thread_turns(...)`
