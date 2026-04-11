# Codex 源码拆解 17：turn materialization 与 pending-request 链

## 这篇看什么

回答：**一个 turn 在 app-server 里到底怎么“成形”？以及 client approval / user-input 这类 pending server request 是怎么存、怎么 replay、怎么 resolve、怎么 abort 的？**

## 先给结论

这一层已经能看到 Codex app-server 很明确的工程分层：

1. `ThreadState` 负责积累 turn 局部状态
2. `bespoke_event_handling` 负责把 runtime event 变成客户端看到的 turn 状态变化
3. `OutgoingMessageSender` 负责 pending server request 的 callback 存储 / replay / abort
4. listener task 负责保证这些动作与事件流顺序一致

一句话：

> turn materialization 不是一次性拼对象，而是 `event accumulation + finalization + replay-safe request lifecycle` 的组合。

## 关键代码路径

- `codex-rs/app-server/src/thread_state.rs`
- `codex-rs/app-server/src/bespoke_event_handling.rs`
- `codex-rs/app-server/src/outgoing_message.rs`
- `codex-rs/app-server/src/codex_message_processor.rs`

## 先看 turn 是怎么积累出来的

### 1. listener 每次先 `track_current_turn_event`
`ThreadState::track_current_turn_event()` 会：

- 把 `TurnStarted.started_at` 填进 `turn_summary.started_at`
- 把 event 喂给 `current_turn_history.handle_event(event)`
- 如果收到 `TurnAborted` / `TurnComplete` 且当前没有 active turn 了，就 reset

这说明 app-server 自己维护了一份：

- 活跃 turn 的临时历史
- turn 级摘要状态

### 2. `active_turn_snapshot()` 是给运行中 turn 用的
运行中 resume / listener 通知时，app-server 可以从 `current_turn_history.active_turn_snapshot()` 拿到当前还没落盘完成的 turn。

这一步非常重要，因为 rollout 文件里的 turn 很可能还不完整。

## turn started / completed / interrupted 是怎么投影的

### TurnStarted
在 `apply_bespoke_event_handling()` 里：

- 先 `abort_pending_server_requests()`
- `thread_watch_manager.note_turn_started(...)`
- 给 V2 client 发 `TurnStarted`

而且发通知时优先取 `active_turn_snapshot()`；如果拿不到，再 synthesize 一个最小 `Turn(status=InProgress)`。

这说明：

> app-server 对客户端的 running-turn 视图，优先相信自己的 in-memory active turn，而不是等 rollout 落盘。

### TurnComplete
`handle_turn_complete()` 会检查 `thread_state.turn_summary.last_error`：

- 有错误 → `TurnStatus::Failed`
- 无错误 → `TurnStatus::Completed`

然后统一走 `emit_turn_completed_with_status()` 发 `TurnCompletedNotification`。

### TurnInterrupted
`handle_turn_interrupted()` 直接发：

- `TurnStatus::Interrupted`
- `error = None`

### Error vs StreamError
这里有个细分值得记：

- `EventMsg::Error`
  - 会更新 `turn_summary.last_error`
  - 是影响最终 turn 状态的
- `EventMsg::StreamError`
  - 只发 error notification（`will_retry: true`）
  - 不直接决定最终 turn 失败

所以不是所有 error-like event 都会进最终 turn 状态。

## running-thread resume 时，turn 是怎么 materialize 的

running-thread resume 不是简单把 rollout 里的 turns 全量返回。

`handle_pending_thread_resume_request()` 会做：

1. 从 `ThreadState.active_turn_snapshot()` 取活跃 turn
2. 从 rollout 读取历史 turns
3. `merge_turn_history_with_active_turn()`
   - 同 turn_id 的 persisted turn 先删掉
   - active turn 再塞进去
4. 根据 `ThreadWatchManager.loaded_status_for_thread()` 算 thread status
5. 如果 thread 实际不 active，把 stale `InProgress` turn 改成 `Interrupted`
6. 返回 `ThreadResumeResponse`

这里的核心判断很稳：

> persisted rollout 是历史底座，active turn snapshot 是热态补丁。

## pending server request 是怎么存的

`OutgoingMessageSender::send_request_to_connections()`：

- 分配一个递增 `RequestId`
- 存 `PendingCallbackEntry { callback, thread_id, request }`
- 然后把 request 发给目标 connection(s)

关键点：

- 这里不仅存 callback
- 还把 `thread_id` 和原始 `request` 一起存了

这就是后面能做：
- `pending_requests_for_thread()`
- `replay_requests_to_connection_for_thread()`
- `cancel_requests_for_thread()`

的前提。

## pending request 怎么 replay

`pending_requests_for_thread()` 会：

- 按 `thread_id` 过滤 pending requests
- 再按 request id 排序

`replay_requests_to_connection_for_thread()` 会：

- 把这些 request 重发给新连接

而 running-thread resume 的顺序是：

1. 先发 `ThreadResumeResponse`
2. 再 replay pending requests
3. 最后再把 connection 正式挂回 thread 订阅列表

这个顺序很专业。

它避免了：
- 连接还没看到历史快照
- 就开始收到 live request / live event

## pending request 怎么 resolve

不是所有 resolve 都直接在 request handler 里做。

很多地方会先调：

- `resolve_server_request_on_thread_listener(...)`

它做的是：

1. 往 listener command channel 发 `ResolveServerRequest`
2. 等 listener 处理完成

listener 收到后再：

- 发 `ServerRequestResolvedNotification`

这个设计的真正目的不是“代码好看”，而是：

> 保证 request resolved 的通知顺序，和 thread event / request 本体顺序一致。

否则就会出现客户端看到：
- resolved 先到了
- 相关 event 还没到

这种时序错乱。

## pending request 怎么 abort

turn 状态切换时，thread-scoped outgoing 会：

- `abort_pending_server_requests()`
- 本质是 `cancel_requests_for_thread(thread_id, error)`

而且这个 error 里会带 reason：
- `turnTransition`

当前能看到至少这些点会触发 abort：

- `TurnStarted`
- `TurnComplete`
- `TurnAborted`
- thread unload / shutdown

这说明 app-server 有一个很强的约束：

> per-thread server request 只能活在当前 turn 语义上下文里，turn 变了就不能悬着。

## 一个很关键但容易忽略的判断

### `TurnCompletedNotification` 本身不携带完整 items
它主要带的是：
- metadata
- status
- error

不是整个 final turn items 列表。

这意味着客户端完整看到的 turn，一般来自两部分：

1. 增量 item / delta / notification 流
2. resume/read 时的 thread/turn 物化结果

也就是说，turn completion notification 更像“状态封口”，不是“完整快照传输”。

## 当前边界判断

- `ThreadState`：turn 局部累积与 active turn snapshot
- `handle_turn_complete` / `handle_turn_interrupted`：turn 终态封口逻辑
- `OutgoingMessageSender`：pending request registry + callback store + replay/cancel
- listener command channel：保证 resolve/resume 与事件流排序一致

## 设计判断

### 1. app-server 把“turn 物化”和“request 生命周期”分开了
这很对。

不然 approval/user-input 这种请求态，会把 turn 状态机搞乱。

### 2. replay 机制说明它真的在认真处理断连/重连
这不是 demo 级设计，而是面向真实长会话的恢复策略。

### 3. listener command channel 是整个时序正确性的关键点
这条线如果没有，很多 request resolved / running resume 的顺序都会乱。

## 还值得继续观察的点

1. 不是所有 request 类型都明显走了 `ServerRequestResolved`，这里可能还有不完全一致的地方
2. `TurnCompletedNotification` 为什么不直接带完整 final items，后续若写 guide 需要单独说明
