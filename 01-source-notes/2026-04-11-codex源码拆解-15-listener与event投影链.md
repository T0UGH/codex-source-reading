# Codex 源码拆解 15：listener 与 event 投影链

## 这篇看什么

回答：**`CodexThread` 里的 runtime event，究竟怎么被 app-server 投影成客户端看到的 notification / request / turn 更新？**

## 先给结论

app-server 这条链已经很清楚了：

1. 先把 connection 订阅到 thread
2. 为该 thread 启一个 listener task
3. listener task 持续 `conversation.next_event()`
4. 每个 event 先更新 `ThreadState`
5. 再按当前订阅连接构造 `ThreadScopedOutgoingMessageSender`
6. 然后走 `apply_bespoke_event_handling(...)` 做协议级投影

一句话：

> listener 是 event pump，`bespoke_event_handling` 是 protocol projector。

## 关键代码路径

- `codex-rs/app-server/src/codex_message_processor.rs`
- `codex-rs/app-server/src/thread_state.rs`
- `codex-rs/app-server/src/bespoke_event_handling.rs`
- `codex-rs/app-server/src/outgoing_message.rs`

## 入口：先订阅 connection，再确保 listener 在跑

`ensure_conversation_listener(...)` 里做两件事：

1. `thread_state_manager.try_ensure_connection_subscribed(...)`
2. `ensure_listener_task_running_task(...)`

这两个步骤缺一不可：

- 没有订阅关系，就不知道该把事件发给谁
- 没有 listener task，就没人去 `next_event()`

这说明 listener 不是纯 thread 级资源，而是：

> thread event source + connection subscription registry 的汇合点

## listener task 的主循环

`ensure_listener_task_running_task(...)` 里：

- 如果现有 listener 已经绑定同一个 `CodexThread`，直接 return
- 否则 `thread_state.set_listener(...)`
- 生成：
  - `cancel_rx`
  - `listener_command_rx`
  - `listener_generation`
- 然后起一个 Tokio task 跑 `select!`

`select!` 同时监听三类输入：

1. `cancel_rx`
   - listener 被 supersede 或 thread teardown
2. `conversation.next_event()`
   - runtime 真正吐出的 thread event
3. `listener_command_rx.recv()`
   - 发给 listener 的控制命令，比如 running-thread resume / resolve server request

这非常关键：

**listener 不只是 event forwarder，还是一个串行化的 thread-side control loop。**

## event 先更新 thread-local state，再做协议投影

listener 收到 event 之后，先：

- `thread_state.track_current_turn_event(&event.msg)`
- 读取 `experimental_raw_events`

然后再取：

- `thread_state_manager.subscribed_connection_ids(conversation_id)`
- 构造 `ThreadScopedOutgoingMessageSender`

最后才调：

- `apply_bespoke_event_handling(...)`

顺序不能反。

因为 app-server 需要保证：

- active turn snapshot 是最新的
- raw event opt-in 状态同步
- 当前连接集合正确

所以这条链不是“收到 event 立刻广播”，而是：

> **先做 thread-local accounting，再做 outward projection。**

## `apply_bespoke_event_handling` 才是真正的协议翻译器

这个文件非常大，但角色很明确：

- 输入：core `EventMsg`
- 输出：app-server protocol notification / request / response side-effect

例如：

### TurnStarted
- `outgoing.abort_pending_server_requests()`
- `thread_watch_manager.note_turn_started(...)`
- 发 `ServerNotification::TurnStarted`

### TurnComplete
- abort pending requests
- `note_turn_completed(...)`
- `handle_turn_complete(...)`

### ExecApprovalRequest / ApplyPatchApprovalRequest / RequestPermissions
- 先 `note_permission_requested(...)`
- 再给客户端发 approval request

### RequestUserInput
- `note_user_input_requested(...)`
- 再发 user-input request

### Error
- `note_system_error(...)`
- 再投影成 error notification

### TurnAborted / interrupt 路径
- `note_turn_interrupted(...)`
- 再走 interrupted turn 的协议投影

### ShutdownComplete
- `note_thread_shutdown(...)`

这说明：

**watch status 的推进，并不在 listener loop 本身，而是在 bespoke event handling 里和协议投影一起推进。**

## raw events 是特判分支，不是主输出模型

listener loop 里专门有个分支：

- 如果 event 是 `RawResponseItem`
- 且 `raw_events_enabled == false`
- 就只 `maybe_emit_hook_prompt_item_completed(...)`
- 然后 `continue`

这说明 raw event stream 在 app-server 里不是默认公共 contract。

默认 contract 仍然是更高层、加工过的 typed notification。

这很合理，不然客户端会被底层 item 流淹死。

## ThreadScopedOutgoingMessageSender 的角色

这个对象很值得注意。

它把三件事绑在一起：

- `outgoing`
- `connection_ids`
- `thread_id`

所以它不是裸发消息，而是：

- 只给当前订阅该 thread 的连接发
- request 也记录 thread_id
- turn 状态切换时可以 `abort_pending_server_requests()`

也就是说：

> listener 不直接碰全局 transport，而是通过 thread-scoped sender 发 thread-local 事件。

这是很干净的封装。

## listener command channel：为什么 running-thread resume 不直接响应？

`ThreadListenerCommand` 目前至少有：

- `SendThreadResumeResponse`
- `ResolveServerRequest`

这个设计非常专业。

原因是：

- running thread resume 不能和 listener 正在吐 event 抢顺序
- resolve server request 也需要在 listener 上下文里串行执行，保证和请求本身的相对顺序

所以 app-server 专门把这些“必须与事件流排序一致”的动作，塞回 listener command channel。

这一步很容易被忽略，但它是整条链里最像工程师踩坑后的设计。

## running-thread resume 的关键补丁动作

`handle_pending_thread_resume_request(...)` 会：

1. 从 `ThreadState.active_turn_snapshot()` 拿当前活跃 turn
2. 从 rollout 读取历史 turns
3. 把 active turn merge 回去
4. 再按 `ThreadWatchManager.loaded_status_for_thread(...)` 设置 thread status
5. 发 `ThreadResumeResponse`
6. `replay_requests_to_connection_for_thread(...)`
7. 再把 connection 正式加回 thread

所以 running-thread resume 不是简单“给你个 thread summary”。

它是在 listener 上下文里拼一个：

- 历史 rollout
- 当前活跃 turn
- 当前 watch status
- 当前未决 server requests

组成的恢复态视图。

## 一个更准确的主链图

`CodexThread.next_event()`
→ listener task
→ `ThreadState.track_current_turn_event()`
→ `subscribed_connection_ids()`
→ `ThreadScopedOutgoingMessageSender`
→ `apply_bespoke_event_handling()`
→ app-server protocol notifications / requests / status changes

而控制命令走另一支：

request-side code
→ `ThreadListenerCommand`
→ listener command channel
→ 在 listener 上下文里串行处理

## 设计判断

### 1. listener 是协议边界上的串行化事件泵

它的价值不是“帮忙转发”，而是：

- 保证 thread 事件有单一串行消费点
- 保证 request/response side-effect 和 event 顺序一致

### 2. `bespoke_event_handling` 是 app-server 的真正业务投影层

如果要理解 app-server 对外到底承诺了什么，就要看这里，不是只看 message_processor。

### 3. `ThreadScopedOutgoingMessageSender` 很关键

它把 thread-local 路由、pending request 管理、notification 范围控制收在一起，避免 listener 到处传 connection 细节。

## 当前边界判断

- listener task：thread event pump + ordering point
- `ThreadState`：listener 本地 event-derived state
- `ThreadScopedOutgoingMessageSender`：thread-local transport facade
- `bespoke_event_handling`：core event → app-server protocol projector

## 后续还值得继续挖的点

1. `handle_turn_complete` / `handle_turn_interrupted` 如何构造 turn 粒度最终态
2. request replay / pending callback 的全链路细节
3. raw event contract 和 typed event contract 的长期边界是否还会再收敛
