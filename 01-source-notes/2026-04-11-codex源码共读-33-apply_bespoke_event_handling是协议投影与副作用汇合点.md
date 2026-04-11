---
title: apply_bespoke_event_handling(...) 是协议投影与副作用汇合点
date: 2026-04-11
tags:
  - Codex
  - 源码共读
  - app-server
  - event-projection
  - runtime
source_files:
  - /Users/wangguiping/workspace/codex/codex-rs/app-server/src/bespoke_event_handling.rs
  - /Users/wangguiping/workspace/codex/codex-rs/app-server/src/thread_state.rs
  - /Users/wangguiping/workspace/codex/codex-rs/app-server/src/thread_status.rs
  - /Users/wangguiping/workspace/codex/codex-rs/app-server/src/codex_message_processor.rs
status: done
---

# Codex 源码共读 33：apply_bespoke_event_handling(...) 是协议投影与副作用汇合点

## 这篇看什么

前一篇已经把 listener task 安装器立住了。

那接下来最该问的就是：

> listener 拿到 event 以后，真正“做事”的那层到底是什么？

也就是：
- 为什么 app-server 不直接把 core `EventMsg` 原样往外吐
- turn started / complete / interrupted 为什么能变成另一套更稳定的协议事件
- approval / user input / permission request 为什么看起来像客户端发起的交互，其实仍然是 thread runtime 的一部分
- `ThreadState` 和 `ThreadWatchManager` 到底在哪些节点被推进

这篇只盯一个函数：

- `apply_bespoke_event_handling(...)`

## 先给主结论

如果先只留一句话，我会留这个版本：

> `apply_bespoke_event_handling(...)` 不是简单的 event → notification 映射器，而是 app-server 里真正把 core `EventMsg` 变成“客户端可见状态变化”的汇合层：它一边做协议投影，一边推进 `ThreadState.turn_summary`、`pending_interrupts/pending_rollbacks`、`ThreadWatchManager` 的运行态，还把 approval / user-input 这类客户端交互重新接回 core `Op::*` 主链。

再压缩一点：

> **它是 app-server 的“事件翻译器 + 副作用执行层”。**

这两个词缺一不可。

---

## 第一层：它不是原始 history builder，而是“在 raw history 之上的加工层”

这个函数很容易被看成 event 总入口，
但其实在 listener loop 里，调用它之前已经做过一层关键动作：

- `thread_state.track_current_turn_event(&event.msg)`

也就是说，原始事件已经先进入：
- `current_turn_history`
- `turn_summary.started_at`
- active turn snapshot

之后才轮到 `apply_bespoke_event_handling(...)`。

这说明它不负责“把所有事件原样存起来”，
而是负责：

> **在已经跟上的 thread-local 状态之上，追加 app-server 的协议与运行时语义。**

这是理解它角色的前提。

---

## 第二层：它不是一个单功能函数，而是三类事情揉在一起

我觉得这函数可以粗分成 3 类分支：

### 1. turn / thread 生命周期投影
例如：
- `TurnStarted`
- `TurnComplete`
- `TurnAborted`
- `ShutdownComplete`
- `Error`

### 2. request/approval/user-input 桥接
例如：
- `ApplyPatchApprovalRequest`
- `ExecApprovalRequest`
- `RequestUserInput`
- `ElicitationRequest`
- `RequestPermissions`

### 3. 各类 item/tool/realtime/collab 的展示层投影
例如：
- realtime conversation 系列
- MCP tool call begin/end
- dynamic tool call
- collab begin/end/wait/close/resume
- hook / reasoning / plan delta 等

这说明它其实不是一个单纯“协议 adapter”，
而是：

> **所有 thread event 在进入 app-server 世界前，最后一次被重新解释的地方。**

---

## 第三层：turn 终态不是 terminal event 自己决定的，而是这里通过累计状态补齐的

这是这函数最值的地方之一。

### `TurnStarted`
它会：
- abort pending server requests
- `note_turn_started(...)`
- 发 `TurnStartedNotification`
- 而且优先取 `active_turn_snapshot()`，没有才 synthesize 一个最小 `Turn`

这说明 started turn 的客户端视图不是直接拿 terminal metadata 拼的，
而是优先相信 in-memory active turn。

### `TurnComplete`
它并不是直接把 `TurnCompleteEvent` 翻译一下就结束。

它会先看：
- `thread_state.turn_summary.last_error`

然后决定：
- `Completed`
- 还是 `Failed`

再走 `handle_turn_complete(...)` 发最终 `TurnCompletedNotification`。

### `TurnAborted`
这里也不是简单“通知 aborted”就完了，
而是：
- 先处理 pending interrupt request
- 再 `note_turn_interrupted(...)`
- 再用统一的 completed-turn 通知语义，发 `Interrupted`

也就是说，在 app-server 世界里：

> **turn 的终态不是某个单一事件天然自带的，而是“累计上下文 + terminal event”共同决定的。**

这就是成熟 runtime 常见的 finalize 思路。

---

## 第四层：`turn_summary` 让这层从“翻译器”升级成“状态拼装器”

`turn_summary` 至少承接这些东西：
- `started_at`
- `file_change_started`
- `command_execution_started`
- `last_error`

### 为什么 `last_error` 特别关键
这说明 Error 不是只发个 notification 就完了，
而是会暂存在 `turn_summary` 里，
等到 turn 真正 complete 时再决定：
- 这轮是不是 failed
- 最终带什么 `TurnError`

### 为什么两个 `*_started` set 也关键
它们让 app-server 能区分：
- 这是 file change 相关 item
- 还是 command execution 相关 item

并且避免重复发 `ItemStarted/ItemCompleted`。

这就说明：

> **这层不仅投影事件，还在维护“客户端应该怎么理解这些事件”的中间状态。**

如果没有这层 set，`ExecCommandOutputDelta` 这种共用低层事件，很难被稳定映射成对的 UI item。

---

## 第五层：approval / user-input 不是外围弹窗，而是这条主链里的桥接分叉点

这是和你 Claude Code 那批笔记风格最接近的一个点。

### `ApplyPatchApprovalRequest`
它会：
- `note_permission_requested(...)`
- 发送 server request 给客户端
- 等客户端结果
- resolve pending server request
- drop guard
- 必要时 synthesize terminal item
- 再 submit `Op::PatchApproval` 回 core

### `ExecApprovalRequest`
也是同样模式：
- note permission requested
- 发送请求
- 等待结果
- resolve pending request
- drop guard
- 必要时 synthesize completion
- submit `Op::ExecApproval`

### `RequestUserInput` / `ElicitationRequest` / `RequestPermissions`
同样不是简单“问用户一句”，
而是：
- 发 request
- 等 client 回答
- 再回流成 `Op::*` 重新喂回 core

所以这里真正发生的是：

> **客户端交互被接成了 thread runtime 的一个中间分叉，而不是 UI 外挂。**

这点非常关键。

---

## 第六层：`ThreadWatchManager` 的推进主要就发生在这里

这也解释了为什么前面说 watch manager 和 state manager 要分开。

在这个函数里，能看到大量：
- `note_turn_started`
- `note_turn_completed`
- `note_turn_interrupted`
- `note_permission_requested`
- `note_user_input_requested`
- `note_system_error`
- `note_thread_shutdown`

也就是说，`ThreadWatchManager` 并不是 listener loop 顺手维护的，
而是在这层 event 投影时被正式推进。

所以更准确地说：

- listener loop 负责读事件并维护 raw/local turn state
- `apply_bespoke_event_handling(...)` 负责把这些事件投影成外显 watch state

这也是它为什么不是简单 adapter 的原因。

---

## 第七层：它会合成 item，不只是转发 item
这层特别值。

例如：
- command approval 可能在 approval 结果出来前先 synthesize 一个 command-execution item
- patch/file-change 也会自己管理 started/completed 的发射时机
- collab 事件会被统一压成 `ThreadItem::CollabAgentToolCall`
- guardian 事件也会被包成另一套 UI item 语义

这说明 app-server 对客户端提供的并不是 core 原始 item 模型，
而是：

> **经过一次“更适合客户端展示和交互”的协议归一化后的 item 模型。**

这一层就是这个函数在做的。

---

## 第八层：它其实在做“错误分类”，而不是所有 error 一视同仁

这里最典型的是：

### `Error`
- 可能更新 `last_error`
- 可能影响最终 turn 状态
- 可能走 rollback-failed 特殊路径

### `StreamError`
- 只发 `will_retry = true`
- 不进入 turn final error

这说明在 app-server 看来：

> **不是所有 error-like event 都有资格成为 turn 终态的一部分。**

这就是成熟 runtime 和“见 error 就标红”的区别。

---

## 这个函数在总链里的真正角色，我会怎么定义

如果要用一句最短的话来讲，我会写：

> **`apply_bespoke_event_handling(...)` 是 app-server 世界里把 core thread event 重新解释成“客户端可交互状态”的中心层：它既翻译协议，又维护中间状态，又把客户端决策桥接回 core。**

它不是：
- 纯 notification mapper
- 纯状态机
- 纯 request bridge

而是三者的交汇点。

---

## 为什么这层设计是对的

### 1. 把协议投影和 raw history 分开
这样 active turn/history 不会被客户端协议形状污染。

### 2. 把 approval / user-input 桥接放在这一层
这样客户端交互始终仍然属于 thread runtime 主链，而不是外围 side channel。

### 3. 用 `turn_summary` 做中间累积
这样 turn 终态可以更稳定、更有语义，而不是盯着 terminal event 单点判断。

### 4. 用 watch manager 做 outward projection
这样“内部发生了什么”和“客户端看到什么状态”能解耦。

---

## 继续细拆的话，下一篇最自然的点

如果按这种风格继续下去，最自然的下一篇是：

- `handle_turn_complete(...)`
  - turn finalization 到底怎么用累计状态组装最终 completed turn

或者：

- `on_command_execution_request_approval_response(...)`
  - 为什么 command approval 不是一个简单 allow/deny 布尔流
