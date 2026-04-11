---
title: resolve_pending_server_request(...) 是 resolved 通知的有序发射器
date: 2026-04-11
tags:
  - Codex
  - 源码共读
  - app-server
  - request-lifecycle
  - ordering
source_files:
  - /Users/wangguiping/workspace/codex/codex-rs/app-server/src/codex_message_processor.rs
  - /Users/wangguiping/workspace/codex/codex-rs/app-server/src/thread_state.rs
  - /Users/wangguiping/workspace/codex/codex-rs/app-server/src/outgoing_message.rs
status: done
---

# Codex 源码共读 40：resolve_pending_server_request(...) 是 resolved 通知的有序发射器

## 这篇看什么

前面已经知道：
- 某些 approval / user-input / elicitation request 发给客户端之后
- 客户端回结果，不会直接在原回调里发 resolved
- 而是先走 `ResolveServerRequest` command

那更细的问题就是：

> 这个 resolved 到底是怎么被发出来的？它为什么值得单独做成一个函数？

这篇只讲：

- `resolve_pending_server_request(...)`

## 先给主结论

如果先留一句话，我会写：

> `resolve_pending_server_request(...)` 的真正价值不在业务复杂度，而在时序位置：它把“某个 server request 已经解决”这件事，重新包装成 listener 上下文里的 thread-scoped notification，只发给当前这个 thread 的订阅者，从而让 resolved 本身也成为 thread 事件流里一个有顺序语义的组成部分。

再压缩一点：

> **它不是 resolved 状态计算器，而是 resolved 时序发射器。**

---

## 第一层：这函数几乎不做逻辑判断，说明它的价值在别处

函数做的事非常少：
1. 取当前 thread 的 subscribed connection ids
2. 新建 `ThreadScopedOutgoingMessageSender`
3. 发 `ServerRequestResolvedNotification`

仅此而已。

这就说明，它真正的价值不是复杂逻辑，
而是：

> **它被放在 listener command 链里的位置。**

和前面 `handle_thread_listener_command(...)` 一样，
这是一个“看起来简单但时序地位很高”的函数。

---

## 第二层：resolved 不是发给全局，而是发给 thread-scoped subscribers

这里它不会 global broadcast，
而是：
- `subscribed_connection_ids(conversation_id)`
- 再包成 `ThreadScopedOutgoingMessageSender`

这说明 app-server 对 resolved 的理解不是：
- 某个客户端请求结束了

而是：
- **这个 thread 上一个 server request 的状态变了**

所以 resolved 本身也被纳入 thread scope。

这点很重要，因为它把 request lifecycle 明确挂到了 thread 维度，
而不是全局无主对象。

---

## 第三层：为什么不在 response handler 里直接发

真正关键点不在函数体，而在它的调用方式：

- 外部回调先调 `resolve_server_request_on_thread_listener(...)`
- 这个 helper 再把命令塞进 listener command channel
- listener 再回到这里发 resolved

也就是说，作者明确拒绝这种做法：
- 在客户端回复回调里，直接 somewhere 发 resolved

为什么？

因为那样会破坏一个关键语义：

> **resolved 必须和 thread 自己的事件流在同一个顺序世界里出现。**

这才是它存在的原因。

---

## 第四层：它是“request sent / request resolved”配对里的后半段

前半段大概是：
- `ThreadScopedOutgoingMessageSender::send_request(...)`

后半段就是它：
- `resolve_pending_server_request(...)`

前者：
- 把 request 发给 thread 相关连接
- 并把 callback / request / thread_id 存进 pending registry

后者：
- 在正确的 listener 顺序点发 `ServerRequestResolved`

所以如果把 request lifecycle 看成一对事件：

1. request sent
2. request resolved

这个函数就是第二个点的有序发射器。

---

## 第五层：它其实也在服务 reconnect 语义

因为 `ThreadScopedOutgoingMessageSender` 和 pending request replay 机制是 thread-scoped 的，
这说明 resolved notification 也天然属于：
- 某个 thread 的 pending interaction 世界

所以把它做成 thread-scoped notification，有个隐含好处：

> reconnect / replay / live subscribe 的模型都更统一。

这也是为什么它不应该是个纯 connection-local callback side effect。

---

## 一句角色定义

如果按源码共读风格压一句话，我会这么写：

> **`resolve_pending_server_request(...)` 的本质不是计算某个 request 是否完成，而是把“完成”这件事，以 thread-scoped 且 listener-ordered 的方式发射回 app-server 事件流。**

---

## 为什么这层设计是对的

### 1. 把 resolved 也纳入 thread ordering
这是最核心的价值。

### 2. 用 thread-scoped sender，而不是全局 sender
保证它只影响当前 thread 的观察者。

### 3. 让 request lifecycle 对 reconnect/live fanout 更一致
这让 request 不会变成 thread 世界之外的异物。

---

## 继续细拆的话，下一篇最自然的点

如果继续往下拆，最自然的是：

1. `OutgoingMessageSender::send_request_to_connections(...)`
2. `replay_requests_to_connection_for_thread(...)`
3. `cancel_requests_for_thread(...)`

这 3 个加起来，基本就能把 pending request 生命周期彻底讲透。
