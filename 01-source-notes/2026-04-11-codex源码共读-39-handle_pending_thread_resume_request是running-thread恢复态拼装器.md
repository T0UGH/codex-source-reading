---
title: handle_pending_thread_resume_request(...) 是 running-thread 恢复态拼装器
date: 2026-04-11
tags:
  - Codex
  - 源码共读
  - app-server
  - resume
  - listener
  - thread-state
source_files:
  - /Users/wangguiping/workspace/codex/codex-rs/app-server/src/codex_message_processor.rs
  - /Users/wangguiping/workspace/codex/codex-rs/app-server/src/thread_state.rs
  - /Users/wangguiping/workspace/codex/codex-rs/app-server/src/thread_status.rs
  - /Users/wangguiping/workspace/codex/codex-rs/app-server/src/outgoing_message.rs
status: done
---

# Codex 源码共读 39：handle_pending_thread_resume_request(...) 是 running-thread 恢复态拼装器

## 这篇看什么

前一篇已经把 `handle_thread_listener_command(...)` 的角色立住了：

- listener command 不是普通 helper 调用
- 而是必须回到 listener 上下文里处理的 ordering-sensitive 动作

那接下来最自然的问题就是：

> 其中最关键的 `SendThreadResumeResponse`，到底是怎么把一个还在运行的 thread 恢复态拼出来的？

也就是：
- 为什么 resume running thread 不能只读 rollout summary
- active turn snapshot 为什么一定要 merge 回来
- 为什么 response / replay pending requests / attach subscription 这个顺序不能换
- “atomically subscribe for new updates” 在代码里到底是怎么近似实现的

这篇只讲：

- `handle_pending_thread_resume_request(...)`

## 先给主结论

如果先留一句话，我会写：

> `handle_pending_thread_resume_request(...)` 不是简单把 thread summary 回给客户端，而是在 listener 上下文里拼一个“恢复态视图”：它会把 rollout 里的持久 turns、listener 维护的 in-memory active turn、watch manager 里的 live status 重新对齐，先回一份当前最可信的 `ThreadResumeResponse`，再 replay 这个 thread 上尚未解决的 server requests，最后才把 connection 正式挂回 live fanout。这条顺序本质上是在模拟“先补齐恢复基线，再接回实时世界”。

再压缩一点：

> **它不是 resume response sender，而是 running-thread reconnect baseline composer。**

---

## 第一层：它不是从空开始组装，而是从一个“已加载 summary”继续修补

这个函数拿到的不是 thread id 和 path 这么原始的输入，
而是一个 `PendingThreadResumeRequest`，里面已经有：
- `request_id`
- `rollout_path`
- `config_snapshot`
- `thread_summary`

也就是说，在进入这个函数之前，外层已经先做了：
- load thread summary
- load config snapshot
- 找到 listener command channel
- 把命令丢进 listener

所以这个函数不是“整个 resume 逻辑”的起点，
而是：

> **running-thread resume 的 listener-side拼装阶段。**

这也解释了为什么它重点不在 path 查找，而在状态对齐。

---

## 第二层：它先读 `active_turn_snapshot()`，说明 running thread 的恢复永远不能只信 rollout

函数一上来先：
- 锁 `thread_state`
- 取 `active_turn_snapshot()`

这一步已经把它和普通“读取一个静态 thread”区分开了。

因为：
- rollout 是持久 history
- 但当前 thread 还在跑
- active turn 可能还没完全落盘

所以这一步表达的核心态度是：

> **running-thread resume 必须把 listener 已经知道、但 rollout 还未完整表达的活跃状态补回来。**

这就是为什么它不是简单的 `load_thread_summary_for_rollout(...)` 的后续一行代码。

---

## 第三层：`has_live_in_progress_turn` 是对 watch-state 滞后的一次防守性修正

函数不会只看 `thread_watch_manager.loaded_status_for_thread(...)`，
而是先算一个：
- `has_live_in_progress_turn`

它来自两路：
1. `conversation.agent_status() == Running`
2. `active_turn.status == InProgress`

这个组合很关键。

因为单看 watch-manager 可能会有滞后窗口，
而单看 active turn snapshot 也未必代表整个 runtime 还在运行。

所以这里其实是在做：

> **runtime 活跃性双重交叉校验。**

这和前面 `resolve_thread_status(...)` 那个“宁可把真实活跃线程看成 Active，也不要误判成 Idle/NotLoaded”的思路是完全一致的。

---

## 第四层：真正的核心是 `populate_thread_turns(...) + active turn merge`

真正重头戏是这一步：
- 从 rollout 读 turns
- 再把 active turn merge 回去

`populate_thread_turns(...)` 干的是：
1. 从 rollout path 读 item
2. `build_turns_from_rollout_items(...)`
3. 如果有 `active_turn`，就 `merge_turn_history_with_active_turn(...)`

而 merge 的规则非常直接：
- 先删掉同 id 的旧 turn
- 再把 active turn push 回去

这说明函数的心智非常清晰：

> **persisted turn 是历史底座，active turn 是热态修补层。**

它不是试图做字段级 merge，
而是直接承认：
- 对于同一个 turn id，当前 in-memory active 版本更可信

这是一种很干净的选择。

---

## 第五层：恢复 running thread 时，还要把“假活跃”的 stale in-progress turn 清成 interrupted

这一步特别值。

函数拿到 thread history 之后，还会：
- 读 `thread_watch_manager.loaded_status_for_thread(...)`
- 调 `set_thread_status_and_interrupt_stale_turns(...)`

这个 helper 的逻辑是：
1. 先用 `resolve_thread_status(...)` 得到最终 thread status
2. 如果最终 status 不是 `Active`
   - 那么所有 lingering `InProgress` turn 都改成 `Interrupted`

这说明作者很清楚一种常见脏状态：

- history 里还留着 `InProgress`
- 但 thread 实际上已经不在活跃态了

所以恢复时不能把这种 stale in-progress turn 直接暴露给客户端，
而必须规范化成 interrupted。

这一步就是把：
- history realism
- 当前 runtime realism

重新对齐。

---

## 第六层：response / replay / attach 的顺序，是这函数最值得学习的地方

这里顺序是：

1. `send_response(ThreadResumeResponse)`
2. `replay_requests_to_connection_for_thread(...)`
3. `try_add_connection_to_thread(...)`

这三个动作如果只看表面，很容易觉得都差不多。

其实不是。

### 为什么先发 response
客户端必须先知道：
- 当前 thread 长什么样
- turns 到什么状态
- 当前 config snapshot 是什么

### 为什么接着 replay pending server requests
如果这个 thread 上已经有：
- approval request
- user input request
- 其他 unresolved server request

那么 reconnecting client 不能只拿到 thread history，
还必须知道当前有哪些 pending interaction 还悬着。

### 为什么最后才 attach subscription
如果先 attach，再 replay：
- reconnecting client 可能会先收到新的 live event
- 再收到旧 pending request

顺序就乱了。

所以这条顺序真正想表达的是：

> **先把恢复基线补齐，再让你重新进入实时广播。**

这就是代码里所谓“atomically subscribe for new updates”的近似实现。

严格说它不是事务级原子，
但在 listener 串行上下文里，这已经是非常好的工程近似。

---

## 第七层：它为什么必须在 listener 上下文里做

如果这函数放在外面执行，会出现至少几个风险：

1. 还没拿到最新 active turn snapshot，就开始回 response
2. response 和 live event 可能交错
3. pending request replay 和 live fanout 可能交错
4. attach subscription 的时点可能早于 replay 完成

而放在 listener 里后，虽然 `tokio::select!` 仍然可能让 event/command 竞争 ready 时机，
但至少有一个关键保证：

> **这些动作都共享同一个 thread-local 串行执行点。**

这就是它能成立的根本。

---

## 第八层：这个函数本质上在做“running thread 的恢复态规范化”，而不是单纯“恢复”

它做的不是 naive restore，
而是带规范化判断的恢复：

- merge active turn
- 根据 live/runtime status 修 thread status
- 必要时把 stale in-progress turn 变 interrupted
- replay unresolved requests
- 再纳入 live subscriptions

所以如果按源码共读风格压一句话，我会写：

> **`handle_pending_thread_resume_request(...)` 的本质不是从磁盘恢复一个 thread，而是把“durable history + listener 本地热状态 + 当前 watch 状态”规范化成一份 reconnect 可用的运行中 thread 视图。**

---

## 为什么这层设计是对的

### 1. 它承认 running thread 和 archived thread 不是一回事
run 中 thread 有热状态，不能只读 rollout。

### 2. 它承认恢复不是只补 transcript，还要补 pending interaction context
所以 replay pending requests 被正式纳入流程。

### 3. 它把 reconnect 顺序做得很讲究
response → replay → attach 的顺序非常成熟。

### 4. 它把 status 和 history 对齐
避免 stale in-progress turn 这种客户端看起来很怪的状态。

---

## 继续细拆的话，下一篇最自然的点

最自然继续拆的是：

1. `populate_thread_turns(...)`
2. `set_thread_status_and_interrupt_stale_turns(...)`
3. `replay_requests_to_connection_for_thread(...)`

因为这三个基本就是这个函数的三块骨架。
