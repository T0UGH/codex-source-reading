# Codex 源码拆解 16：thread_watch_manager 与 thread_state_manager 的分工

## 这篇看什么

回答：**app-server 里为什么既有 `ThreadStateManager`，又有 `ThreadWatchManager`？这两个到底谁管什么？**

## 先给结论

这两个名字很像，但职责其实不同：

1. `ThreadStateManager`
   - 管 **订阅关系 + listener 生命周期 + 活跃 turn 快照 + pending 操作状态**
   - 更偏“listener 内务状态”
2. `ThreadWatchManager`
   - 管 **线程对外展示状态**
   - 更偏“UI / client 可观察状态”

一句话：

> `thread_state_manager` 管内部协调，`thread_watch_manager` 管对外可见状态。

## 关键代码路径

- `codex-rs/app-server/src/thread_state.rs`
- `codex-rs/app-server/src/thread_status.rs`
- `codex-rs/app-server/src/codex_message_processor.rs`
- `codex-rs/app-server/src/bespoke_event_handling.rs`

## `ThreadStateManager`：它管的是 listener 侧账本

`thread_state.rs` 里的核心结构：

- `live_connections`
- `threads`
- `thread_ids_by_connection`

每个 thread entry 里又有：

- `pending_interrupts`
- `pending_rollbacks`
- `turn_summary`
- `cancel_tx`
- `experimental_raw_events`
- `listener_generation`
- `listener_command_tx`
- `current_turn_history`
- `listener_thread`

这说明它的职责是：

### 1. connection ↔ thread 订阅关系
- 哪个 connection 订了哪个 thread
- 哪些 connection 还活着
- 某个 connection 关了以后哪些 thread 失去订阅者

### 2. listener 生命周期
- 当前 listener 绑的是哪个 `CodexThread`
- cancel / supersede
- generation 防止旧 listener 回收新 listener
- listener command channel

### 3. turn 内局部状态
- 当前 active turn history
- pending interrupt / pending rollback
- turn_summary
- raw event opt-in

也就是说，`ThreadStateManager` 关心的是：

> 这个 thread 的 listener 现在怎么跑、给谁跑、手里攒了什么局部状态。

## `ThreadWatchManager`：它管的是可观察状态机

`thread_status.rs` 里核心状态非常收敛：

- `RuntimeFacts`
  - `is_loaded`
  - `running`
  - `pending_permission_requests`
  - `pending_user_input_requests`
  - `has_system_error`

然后通过 `loaded_thread_status(...)` 投影成：

- `NotLoaded`
- `Idle`
- `Active { active_flags }`
- `SystemError`

而且它还会：

- `send_server_notification(ThreadStatusChanged(...))`
- `subscribe_running_turn_count()`

这说明 watch manager 的语义很纯：

> 把 runtime 事实压成对外可观察的 thread status。

## 两者怎么配合

### 配合点 1：listener 先靠 state manager 找订阅连接
listener loop 每次 event 到来时会：

- `thread_state_manager.subscribed_connection_ids(thread_id)`

这一步是路由能力，watch manager 不参与。

### 配合点 2：event 投影时再推进 watch status
`apply_bespoke_event_handling(...)` 里：

- `TurnStarted` → `note_turn_started`
- `TurnComplete` → `note_turn_completed`
- `Error` → `note_system_error`
- permission/user-input request → `note_permission_requested` / `note_user_input_requested`
- interrupt / shutdown → 对应 watch update

也就是说：

- event 顺序和 listener 归 `state_manager` 体系兜住
- status 对外表达由 `watch_manager` 更新

### 配合点 3：running-thread resume 要同时吃两边信息
`handle_pending_thread_resume_request(...)`：

- 从 `ThreadState.active_turn_snapshot()` 拿活跃 turn
- 从 `ThreadWatchManager.loaded_status_for_thread()` 拿当前对外状态

这特别能说明边界：

- active turn 历史快照是 state 侧东西
- thread 当前展示状态是 watch 侧东西

两者都需要，但不是同一概念。

## 为什么不合并成一个 manager？

从架构角度看，不建议合并。

因为两者变化频率和职责面不同：

### `ThreadStateManager` 更像控制器内务
- connection registry
- listener cancel/generation
- command channel
- active turn buffer
- pending op bookkeeping

### `ThreadWatchManager` 更像 view-model / projection
- 只关心 thread 对外 status
- 把复杂 runtime 状态压成小而稳定的枚举
- 直接服务 UI / 客户端通知

如果硬合并，通常会出现两个坏处：

1. 内务状态污染公共状态机
2. 视图状态变更和 listener 控制逻辑强耦合

现在这种拆法是合理的。

## 一个更准确的比喻

- `ThreadStateManager` = 调度室台账
- `ThreadWatchManager` = 面向外部的大屏状态板

调度室知道：
- 谁在值班
- 哪条线接进来
- 哪个动作还没做完
- 当前活跃 turn 的局部历史

大屏只显示：
- 线程没加载 / 空闲 / 活跃 / 出错
- 是否在等批准 / 等用户输入

## 当前边界判断

- `ThreadStateManager`
  - connection ↔ thread 订阅关系
  - listener lifecycle / generation / cancel
  - active turn snapshot
  - pending interrupt/rollback/request ordering context
- `ThreadWatchManager`
  - 对外线程状态投影
  - running turn 计数
  - thread status changed notification
  - permission/user-input/system-error 等 active flag 映射

## 设计判断

### 1. `ThreadStateManager` 更接近 controller state

它里面大量字段都不是用户关心的最终状态，而是为了把 listener loop 跑对。

### 2. `ThreadWatchManager` 更接近 observable projection

它是给 app-server client/TUI/SDK 消费的稳定状态面。

### 3. 这套拆分是对的

至少从当前代码看，这不是“抽象过度”，而是对复杂事件系统做的必要去耦。

## 后续还值得继续挖的点

1. `turn_summary` 在 complete / interrupted / error 路径下到底如何最终 materialize 成 turn 结果
2. pending interrupt / rollback 这套状态机是否还有历史债
3. running turn count 最终是哪些 client surface 在消费
