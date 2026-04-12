---
title: DynamicToolCall 为什么不走 ServerRequestResolved
date: 2026-04-12
tags:
  - Codex
  - app-server
  - DynamicToolCall
  - ServerRequestResolved
  - boundary-judgment
source_files:
  - /Users/wangguiping/workspace/codex/codex-rs/core/src/tools/handlers/dynamic.rs
  - /Users/wangguiping/workspace/codex/codex-rs/app-server/src/bespoke_event_handling.rs
  - /Users/wangguiping/workspace/codex/codex-rs/app-server/src/dynamic_tools.rs
  - /Users/wangguiping/workspace/codex/codex-rs/app-server/src/outgoing_message.rs
  - /Users/wangguiping/workspace/codex/codex-rs/app-server/tests/suite/v2/dynamic_tools.rs
  - /Users/wangguiping/workspace/codex/codex-rs/app-server/tests/suite/v2/turn_start.rs
status: done
---

# DynamicToolCall 为什么不走 ServerRequestResolved

## 这篇要回答什么

之前最大的疑点是：

> `DynamicToolCall` 没走 `ServerRequestResolved`，到底是“还没迁完”，还是“本来就不该走”？

这次把链路再往前打了一刀后，我的判断已经可以收敛：

> **`DynamicToolCall` 现在更像是有意不走 `ServerRequestResolved`，而不是一个高概率遗漏的未迁移点。**

更准确一点：

> **它在传输实现上复用了 pending server request 的 callback/replay/cancel 机制，但在对外语义上属于 item lifecycle，不属于 server request resolved lifecycle。**

---

## 先给结论

如果只留 4 条：

1. `DynamicToolCall` 发给客户端时，底层确实走了 `OutgoingMessageSender::send_request(...)`，所以它**实现层**属于 pending request 世界。
2. 但它从 core 发出来时，名字就是 `DynamicToolCallRequest` / `DynamicToolCallResponse`，主身份是 `call_id` + `turn_id`，不是 `request_id`。
3. app-server 收到它后，协议投影走的是：
   - `ItemStarted`
   - `item/tool/call`
   - `ItemCompleted`
   而不是 `ResolveServerRequest` → `ServerRequestResolved`。
4. 测试也在验证这条 item lifecycle 主线；和 approval / elicitation 那类显式断言 `serverRequest/resolved` 的测试是两套语义。

所以当前最稳的边界判断不是：

- `DynamicToolCall` 是一个漏接 `ServerRequestResolved` 的坏洞

而是：

- **`DynamicToolCall` 是“借用了 request transport 机制的 item-scoped 交互”，它的完成语义被设计成 `DynamicToolCallResponse` / `ItemCompleted`，不是 `ServerRequestResolved`。**

---

## 证据 1：core 从一开始就把它建模成 call/response，不是 request/resolved

`core/src/tools/handlers/dynamic.rs` 这条链很关键。

它做的事情是：

1. 在 active turn 里登记 pending dynamic tool call
2. 发 `EventMsg::DynamicToolCallRequest`
3. 等客户端返回
4. 再发 `EventMsg::DynamicToolCallResponse`

这里最重要的不是“也有 pending”，而是：

- 发出事件叫 `DynamicToolCallRequest`
- 回来事件叫 `DynamicToolCallResponse`
- 核心标识是 `call_id` / `turn_id`

这说明在 core 自己的世界里，它本来就不是：

- 某个 app-server server request 被 resolved

而是：

- 某个 turn 内的 dynamic tool call 发起并结束

也就是说，它的**第一身份是 turn item / tool call**，不是 request tracking artifact。

---

## 证据 2：app-server 协议投影也把它当 item，而不是 resolved event

`app-server/src/bespoke_event_handling.rs` 的分支已经把这个边界写得很清楚：

### `EventMsg::DynamicToolCallRequest`
它会先发：

- `ItemStartedNotification`
- item 类型是 `ThreadItem::DynamicToolCall`

然后才发真正给客户端的请求：

- `ServerRequestPayload::DynamicToolCall(params)`

### `EventMsg::DynamicToolCallResponse`
它会发：

- `ItemCompletedNotification`
- item 还是 `ThreadItem::DynamicToolCall`

这两步之间，根本没有走：

- `ResolveServerRequest`
- `resolve_pending_server_request(...)`
- `ServerRequestResolvedNotification`

这不是偶然漏了一行代码那么简单。

因为如果作者真把它当“已发出的 server request 等待 resolved”，最自然的写法应该是复用已有的 resolved 通道。

但这里没有。

这里选择的是另一条完整闭环：

- item started
- item/tool/call request
- item completed

所以它看起来更像**有意分叉的协议语义**。

---

## 证据 3：它只复用了 request transport 机制，没有复用 resolved 语义

`app-server/src/outgoing_message.rs` 说明了一个很容易误判的点：

`DynamicToolCall` 虽然不走 `ServerRequestResolved`，但它发送时确实走了：

- `send_request_to_connections(...)`
- `request_id_to_callback`
- `thread_id` 绑定
- `pending_requests_for_thread(...)`
- `replay_requests_to_connection_for_thread(...)`
- `cancel_requests_for_thread(...)`

这意味着：

- **实现层**：它确实是 thread-scoped pending request，可重放、可取消
- **语义层**：它的“完成”不是通过 resolved notification 发射

这正是前面最容易混淆的地方。

所以正确说法不是：

- `DynamicToolCall` 不属于 pending request 世界

而是：

- **它属于 pending request transport 世界，但不属于 `ServerRequestResolved` 这套对外完成语义。**

---

## 证据 4：真正的回包路径是 `Op::DynamicToolResponse`，不是 resolve command

`app-server/src/dynamic_tools.rs` 的 `on_call_response(...)` 把这个问题基本打死了。

客户端对 `item/tool/call` 返回结果后，app-server 做的是：

1. 解码 `DynamicToolCallResponse`
2. 构造 core 需要的 `CoreDynamicToolResponse`
3. 直接 `conversation.submit(Op::DynamicToolResponse { ... })`

这里最关键的是：

- 它把客户端回包直接送回 core 的 dynamic tool 响应入口
- 没有把“回包成功”先转成 `ResolveServerRequest`

而 approval / elicitation 那类链路，恰恰会把“客户端已答复”重新安排到 listener command 顺序里，再发 `ServerRequestResolved`。

`DynamicToolCall` 没这么做，说明设计者认为：

- 它的业务完成语义已经由 `DynamicToolResponse` + 后续 `DynamicToolCallResponse` 事件承担
- 没必要再附送一个并行的 resolved 语义

---

## 证据 5：测试在验证 item lifecycle，不在验证 resolved lifecycle

`app-server/tests/suite/v2/dynamic_tools.rs` 这条测试主线验证的是：

1. `ItemStarted`
2. 收到 `ServerRequest::DynamicToolCall`
3. 客户端返回 `DynamicToolCallResponse`
4. `ItemCompleted`
5. 后续模型收到了 function_call_output

这里没有断言：

- `serverRequest/resolved` 必须出现
- `resolved` 必须在 turn completed 前出现

反过来，`app-server/tests/suite/v2/turn_start.rs` 对 approval / file-change 等 request 的测试，明确会断言：

- `serverRequest/resolved` 出现
- 而且必须早于 `turn/completed`

这说明测试作者眼里，这两类东西本来就是两套协议契约。

如果 `DynamicToolCall` 只是漏迁，测试通常最容易暴露这类不一致；但这里测试结构本身就在强化“它走 item lifecycle”。

---

## 一个容易误判的点：为什么它明明是 request，却不走 resolved

因为这里有两层“完成”语义：

### 第一层：transport / callback 完成
客户端对 `item/tool/call` 返回 JSON-RPC response。

这一层确实是 request-response。

### 第二层：产品语义完成
某个 turn 里的 dynamic tool call 已经完成，并以结果 item 形式回写到线程/模型链路中。

这一层在当前设计里由：

- `DynamicToolCallResponse`
- `ItemCompleted`
- function_call_output 回写模型

来表达。

`ServerRequestResolved` 更像另一类协议语义：

- “这个 thread-scoped server request 现在被顺序化解决了”

这套语义更适合：

- approval
- elicitation
- user input
- file change / patch 这类需要显式回答“请求状态何时 resolved”的交互

而 `DynamicToolCall` 的读者真正关心的是：

- tool call 开始了没有
- tool call 结果是什么
- 最后有没有回进模型继续跑

所以它被折叠进 item lifecycle，是说得通的。

---

## 当前最稳的边界判断

### 不是“未迁移洞”
到目前证据为止，我不再推荐把 `DynamicToolCall` 定义成：

- “最可疑的未迁移到 `ServerRequestResolved` 的点”

这个判断现在已经太弱了。

### 更准确的说法
我现在会把它写成：

> **`DynamicToolCall` 复用了 thread-scoped pending request 的发送/重放/取消机制，但它的对外完成语义被设计成 item lifecycle（`ItemStarted` / `ItemCompleted` + `DynamicToolCallResponse`），而不是 `ServerRequestResolved`。**

再压缩一点：

> **它是 transport 上的 request，语义上的 item。**

这句话我认为现在已经足够稳。

---

## 这对 guidebook 写法意味着什么

后面如果在 guidebook 正文里提这个点，建议不要写成：

- `DynamicToolCall` 还没接上 `ServerRequestResolved`

更稳的写法应该是：

- `DynamicToolCall` 看起来和其他 thread-scoped client request 共用底层 request registry，但它在协议语义上被单独建模成 item/tool/call 生命周期，因此不会通过 `ServerRequestResolved` 对外宣告完成

这样不会把“实现复用”和“协议语义”混成一件事。

---

## 还剩什么边角问题

这个点主体已经收敛了，只剩两个边角值得留意：

1. 未来作者会不会为了统一观察面，再给 `DynamicToolCall` 补一层 resolved-style 辅助通知
2. 除 `DynamicToolCall` 之外，legacy approval request 那些边角形态里，是否还有真正的“半迁移”残留

但至少 `DynamicToolCall` 本身，我认为已经不该再放在“高度可疑未迁移点”列表里。

---

## 一句话拍板

> **`DynamicToolCall` 不是没迁到 `ServerRequestResolved`；它是故意绕开这套语义，改走 item lifecycle。真正复用的只是底层 request transport，不是 resolved 协议语义。**
