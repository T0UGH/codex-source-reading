---
title: handle_turn_complete(...) 是 turn 终态封口器
date: 2026-04-11
tags:
  - Codex
  - 源码共读
  - app-server
  - turn
  - finalization
source_files:
  - /Users/wangguiping/workspace/codex/codex-rs/app-server/src/bespoke_event_handling.rs
  - /Users/wangguiping/workspace/codex/codex-rs/app-server/src/thread_state.rs
status: done
---

# Codex 源码共读 37：handle_turn_complete(...) 是 turn 终态封口器

## 这篇看什么

前面已经知道：
- turn started / error / item state 都会先进入 thread-local 累积
- `apply_bespoke_event_handling(...)` 会在 terminal event 到来时做协议投影

但更细的问题是：

> **一轮 turn 到底是在哪个点真正被“封口”的？**

以及：
- 为什么 `TurnCompleteEvent` 自己不直接决定成功或失败
- `started_at` / `completed_at` / `duration_ms` 分别来自哪里
- 为什么 `turn_summary` 会在这里被一次性 take 掉

这篇只讲：

- `handle_turn_complete(...)`

## 先给主结论

如果先留一句话，我会写：

> `handle_turn_complete(...)` 的真正职责不是“收到 TurnComplete 就发通知”，而是把这一轮在运行中不断累积的 turn-local 状态正式封口：它把 `ThreadState.turn_summary` 一次性取出来，用里面的 `last_error` 决定这轮最终到底是 `Completed` 还是 `Failed`，再把先前在 `TurnStarted` 时记录下来的 `started_at` 和 terminal event 提供的 `completed_at/duration_ms` 拼成一条最终的 `TurnCompletedNotification`，同时完成对这轮 `turn_summary` 的清空切换。

再压缩一点：

> **它不是 terminal event 的转发器，而是 turn 累积状态的收口器。**

---

## 第一层：这个函数最关键的输入不是 `TurnCompleteEvent`，而是 `turn_summary`

乍一看它好像只是处理一个完成事件，
但真正决定它价值的是另一个输入：

- `turn_summary`

也就是之前运行中积累出来的：
- `started_at`
- `last_error`
- 一些 item start/completion 相关 set

这说明在 Codex 里，turn 终态不是：
- 某个 terminal event 单点决定

而是：
- 运行过程中积累出的上下文
- 在 terminal 时刻统一收口

所以这个函数天然就是“finalizer”，不是“event mapper”。

---

## 第二层：成功/失败不是看 `TurnCompleteEvent`，而是看有没有 `last_error`

这是这个函数最值的一点。

它会先把 `turn_summary` 整个拿出来，
然后只做一个关键判断：

- `turn_summary.last_error` 是不是 `Some(...)`

如果有：
- 最终 status = `Failed`
- error 就带进去

如果没有：
- 最终 status = `Completed`

也就是说，`TurnCompleteEvent` 在这里扮演的不是“判决书”，
更像是：
- 一个“这一轮已经到了封口时机”的信号

真正的判决书是：
- 之前累积出来的 `last_error`

这个设计非常成熟。

因为它说明系统不相信 terminal event 本身就能完整表达结果语义，
而是把“是否失败”交给更长时间窗口里的状态积累来决定。

---

## 第三层：时间信息是分三路拼的
这个函数拼最终 completed turn 时，时间字段不是来自同一个地方：

### `started_at`
来自：
- `turn_summary.started_at`
- 而这个字段又是在 `TurnStarted` 时由 `track_current_turn_event(...)` 提前记下来的

### `completed_at`
来自：
- `TurnCompleteEvent.completed_at`

### `duration_ms`
来自：
- `TurnCompleteEvent.duration_ms`

这说明 `handle_turn_complete(...)` 的真正工作不是把一个完整对象往外抄，
而是：

> **把“开始时刻的本地累计”和“结束时刻的 terminal metadata”拼成一个最终 turn。**

这也是它为什么必须存在的原因。

---

## 第四层：`find_and_remove_turn_summary(...)` 是这函数真正的“封口动作”

这个 helper 的核心只有一件事：
- `std::mem::take(&mut state.turn_summary)`

这一步意义非常大。

因为它不是读一眼 summary，
而是：

> **把这一轮的 summary 从 thread state 里完整拿走，并把 slot 重置成默认空状态。**

所以 `handle_turn_complete(...)` 不只是决定通知内容，
还承担了：
- turn-local 临时状态生命周期结束
- 为下一轮 turn 清场

这就是真正的 finalization。

---

## 第五层：为什么这里必须是 destructive read，而不是只读不清

如果这里只是读：
- `state.turn_summary`

不清掉，就会有两个明显风险：

1. 下一轮 turn 会继承上一轮的 `last_error`
2. `started_at`、item-start sets 等也可能串 turn 泄漏

所以这里用 `mem::take(...)` 很对，
因为它在语义上明确表达：

> **这一轮 turn 的 summary 到这里已经消费完了。**

这和很多 runtime 里 “drain + finalize” 的设计非常像。

---

## 第六层：这函数输出的是最小终态，不负责完整 items 回放
最终 `TurnCompletedNotification` 里，
turn 的 `items` 是空的。

这说明它不负责：
- 再把整轮 items 全量塞回一次

而只负责：
- 最终 status
- error
- started/completed/duration

也就是说，它的定位非常克制：

> **只做 turn terminal status sealing，不做整轮历史重放。**

这点很好。

否则这个函数就会和历史回放/turn item 累积逻辑纠缠在一起。

---

## 第七层：它和 `handle_turn_interrupted(...)` 共享一条输出语义，只是判决不同
`handle_turn_interrupted(...)` 最终也走同样的 completed-turn 输出路径，
只是 status 固定为：
- `Interrupted`

所以从协议角度看，Codex app-server 更偏向：

- “一轮结束了” → 都进 `TurnCompletedNotification`
- 区别只是 `status` 是什么

这很说明问题。

它表明 app-server 在 turn 这层的最终对外抽象是：

> **turn 结束是一类统一事件，具体是 completed / failed / interrupted 再由 status 区分。**

而 `handle_turn_complete(...)` 就是正常终态里的那条封口路径。

---

## 第八层：这个函数其实把“Error event”和“TurnComplete event”重新绑定回同一条最终语义上

如果没有这个函数，系统可能会变成：
- Error event 单独红
- TurnComplete event 单独绿
- 客户端自己猜这一轮到底算失败还是成功

现在不是。

现在的路径是：
1. Error event 到来时把 `last_error` 暂存
2. TurnComplete 到来时统一判断
3. 最终只通过一个 terminal turn status 告诉客户端这轮算什么

这是一种明显更稳的设计。

> 它把分散事件重新绑定成一个更稳定的终态语义。

---

## 这个函数在总链里的角色，我会怎么定义

如果按源码共读风格压一句话，我会这么写：

> **`handle_turn_complete(...)` 不是完成事件的转发层，而是 app-server 里把“这一轮运行中积累出来的 turn-local 事实”封成最终 completed/failed turn 语义的最后一刀。**

这个定义我觉得非常贴切。

---

## 为什么这层设计是对的

### 1. 把失败判定从 terminal event 里解耦出来
终态由累计状态决定，比单点事件稳。

### 2. 用 `mem::take` 明确 turn-local 生命周期边界
这让 turn summary 不会泄漏到下一轮。

### 3. 只输出最小 terminal metadata，不重放 items
边界很干净。

### 4. 和 interrupted 共享 completed-turn 输出模型
协议对外更稳定。

---

## 继续细拆的话，下一篇最自然的点

最自然继续拆的是：

1. `find_and_remove_turn_summary(...)`
2. `handle_turn_interrupted(...)`
3. `handle_error(...)`

因为这三者和 `handle_turn_complete(...)` 实际上组成了一套完整的 turn finalization 小闭环。
