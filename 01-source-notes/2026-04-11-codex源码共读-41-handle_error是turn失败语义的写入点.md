---
title: handle_error(...) 是 turn 失败语义的写入点
date: 2026-04-11
tags:
  - Codex
  - 源码共读
  - app-server
  - error
  - turn-status
source_files:
  - /Users/wangguiping/workspace/codex/codex-rs/app-server/src/bespoke_event_handling.rs
  - /Users/wangguiping/workspace/codex/codex-rs/app-server/src/thread_state.rs
  - /Users/wangguiping/workspace/codex/codex-rs/protocol/src/protocol.rs
status: done
---

# Codex 源码共读 41：handle_error(...) 是 turn 失败语义的写入点

## 这篇看什么

前面 `handle_turn_complete(...)` 那篇已经知道：
- completed vs failed
- 不是看 `TurnCompleteEvent` 自己
- 而是看 `turn_summary.last_error`

那接下来就要追一个更细的问题：

> 这个 `last_error` 到底是在哪里被写进去的？

这篇只讲：

- `handle_error(...)`

## 先给主结论

如果先留一句话，我会写：

> `handle_error(...)` 本身非常小，但它在 app-server turn finalization 体系里地位很高：它把一个“会影响 turn 终态的错误”从即时 notification 层写入到 `ThreadState.turn_summary.last_error`，从而让这次错误不只是当场被看见，还能在稍后 `TurnComplete` 到来时被解释成真正的 `TurnStatus::Failed`。换句话说，它不是报错器，而是失败语义的持久写入点。

再压缩一点：

> **它不是发 error 的地方，而是让 turn 以后会失败的地方。**

---

## 第一层：这个函数真正值钱的不是函数体，而是它所处的位置

函数体几乎只有：
- lock `thread_state`
- `state.turn_summary.last_error = Some(error)`

看起来非常普通。

但这恰恰说明：

> **它的价值不在逻辑复杂度，而在“这个写动作发生在整个状态机的哪一层”。**

因为一旦这里没写，后面 `handle_turn_complete(...)` 根本不知道这轮该失败。

所以这个函数虽然小，
但它是整个 turn finalization 小闭环里的关键写入点。

---

## 第二层：它不直接决定客户端看到什么，而是先把“失败资格”写进 summary

这是它和普通 notification handler 最大的区别。

它不是：
- 看到 error 就当场给 turn 改状态

而是：
- 先写到 `turn_summary.last_error`
- 等 `TurnComplete` 再由 finalizer 统一收口

这说明 app-server 的失败语义是两段式的：

1. error arrival
2. terminal finalization

而 `handle_error(...)` 就是第 1 段里把“失败资格”保存下来的地方。

---

## 第三层：不是所有 Error 都会走到它这里

这点很重要。

在真正调用 `handle_error(...)` 之前，外层 `EventMsg::Error` 分支会先做两层过滤：

### 1. rollback-failed 特判
如果是：
- `ThreadRollbackFailed`

就走 rollback failure 的专门逻辑，
而不会按普通 turn error 来写 summary。

### 2. `affects_turn_status()` 过滤
如果当前错误不影响 turn status，
就不会调用 `handle_error(...)`。

这说明作者非常明确地区分了：

- 会影响 turn 最终失败语义的错误
- 只是即时错误/辅助错误/不该污染 turn 终态的错误

所以这里不是“Error event 全量入 summary”，
而是：

> **只有通过分类门槛的 error，才会进入 turn failure 累积层。**

这非常重要。

---

## 第四层：它把“即时错误通知”和“最终失败状态”切成了两层

在外层 Error 分支里，通常会有两件事：

1. 发即时 `ErrorNotification`
2. 调 `handle_error(...)` 写 `last_error`

这两个动作不是重复，而是语义不同。

### 即时通知
- 告诉客户端：刚刚发生了一个 error

### 写入 `last_error`
- 告诉系统：这一轮 turn 最后应被视为 failed 的候选状态

这是一种非常成熟的设计。

否则系统很容易陷入：
- 只即时通知，但最后 turn 还显示 Completed
- 或者见 error 就立刻把 turn 打死，没有 terminal 收口

现在这套做法把二者分开了。

---

## 第五层：`last_error` 是单值，不是列表，这意味着“最终失败原因”是最后一次覆盖结果

这也是个很细但很值的点。

`turn_summary.last_error` 是：
- `Option<TurnError>`

不是：
- `Vec<TurnError>`

所以如果一轮里来了多个 turn-affecting error，
更晚的那个会覆盖更早的那个。

这说明 app-server 对 turn finalization 的态度是：

> **最终 turn error 需要一个最终版本，而不是把所有 error 都堆给客户端。**

这是不是最理想，可以讨论；
但从 UI/协议收敛角度看，这是完全能理解的取舍。

---

## 第六层：这个函数其实是 `handle_turn_complete(...)` 的前置写入半边

如果把整个 turn finalization 小闭环拆开：

### 写入半边
- `handle_error(...)`

### 读取/封口半边
- `handle_turn_complete(...)`

一个负责：
- 在运行中写失败候选

一个负责：
- 在结束时读取并封成最终 terminal status

所以这两个函数是一对。

如果单看 `handle_turn_complete(...)`，会觉得为什么这么依赖 `last_error`；
如果单看 `handle_error(...)`，会觉得只是赋值。

只有把它们合起来，才会看出完整设计：

> **error 到来时先写入，turn 结束时再结算。**

这是一套很清楚的状态机模式。

---

## 第七层：为什么不在 `Error` 分支里直接发 `TurnCompleted(Failed)`

这个问题很值得问。

因为如果一见 error 就立即把 turn 判死，至少会有几个问题：

1. turn 还没真的结束
2. 后面可能还有其他 item / delta / cleanup
3. 客户端会提前看到 terminal 语义
4. turn started/completed 生命周期就不完整了

所以现在这套做法的意义非常明确：

> **错误可以先发生，但 terminal 判决必须等 turn 真正 complete。**

这就是 `handle_error(...)` 只写 summary、不直接做终态通知的原因。

---

## 一句角色定义

如果按源码共读风格压一句话，我会这么写：

> **`handle_error(...)` 的真正角色不是转发错误，而是在 app-server 的 turn 状态机里，把一个即时错误登记成“这轮最终可能失败”的持久信号。**

---

## 为什么这层设计是对的

### 1. 把即时错误和最终失败语义分开
这是最核心的价值。

### 2. 只允许会影响 turn status 的错误进入 finalization 链
防止各种辅助错误污染终态。

### 3. 用单值 `last_error` 收敛最终失败原因
对客户端和协议都更稳定。

---

## 继续细拆的话，下一篇最自然的点

最自然继续拆的是：

1. `ErrorEvent::affects_turn_status()`
2. `handle_thread_rollback_failed(...)`
3. `find_and_remove_turn_summary(...)`

因为这三块刚好围在 `handle_error(...)` 周围，能把错误语义的小闭环彻底讲透。
