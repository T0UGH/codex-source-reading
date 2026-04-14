---
title: app-server request shape 分类与收口
date: 2026-04-12
tags:
  - Codex
  - app-server
  - ServerRequestResolved
  - request-lifecycle
  - boundary-judgment
source_files:
  - /Users/wangguiping/workspace/codex/codex-rs/app-server-protocol/src/protocol/common.rs
  - /Users/wangguiping/workspace/codex/codex-rs/app-server/src/bespoke_event_handling.rs
  - /Users/wangguiping/workspace/codex/codex-rs/app-server/src/thread_state.rs
  - /Users/wangguiping/workspace/codex/codex-rs/app-server/src/message_processor.rs
  - /Users/wangguiping/workspace/codex/codex-rs/app-server/tests/suite/v2/dynamic_tools.rs
  - /Users/wangguiping/workspace/codex/codex-rs/app-server/tests/suite/v2/turn_start.rs
status: done
---

# app-server request shape 分类与收口

## 结论

这轮把 `ServerRequest` 枚举里当前可见的 request shape 全盘了一遍。

现在我更愿意直接下这个判断：

> **当前 app-server 的 request shape 基本没有“身份不明”的灰区了。**

可以稳定分成 4 类：

1. **V2 thread-scoped + 已迁入 `ServerRequestResolved`**
2. **V2 thread-scoped + 有意走 item lifecycle，不走 `ServerRequestResolved`**
3. **global/threadless request，本来就不适合进 `ServerRequestResolved`**
4. **deprecated V1 legacy holdout**

换句话说：

> **这条线现在已经从“找最大可疑点”切到“只剩 legacy 尾巴清点”阶段了。**

---

## 1. 先看枚举面：当前总共就这 9 类 request

`app-server-protocol/src/protocol/common.rs` 里，当前 `ServerRequest` 枚举是这 9 个：

### NEW APIs
1. `CommandExecutionRequestApproval`
2. `FileChangeRequestApproval`
3. `ToolRequestUserInput`
4. `McpServerElicitationRequest`
5. `PermissionsRequestApproval`
6. `DynamicToolCall`
7. `ChatgptAuthTokensRefresh`

### DEPRECATED APIs
8. `ApplyPatchApproval`
9. `ExecCommandApproval`

这个枚举面很重要，因为它意味着：

- 我们现在不是在猜“还有多少种 request”
- 而是在给**当前完整集合**做分类

---

## 2. 分类表

| request shape | 当前分类 | 关键理由 |
|---|---|---|
| `CommandExecutionRequestApproval` | 已迁入 `ServerRequestResolved` | V2、thread-scoped、response handler 显式 `resolve_server_request_on_thread_listener(...)` |
| `FileChangeRequestApproval` | 已迁入 `ServerRequestResolved` | 同上 |
| `ToolRequestUserInput` | 已迁入 `ServerRequestResolved` | 同上 |
| `McpServerElicitationRequest` | 已迁入 `ServerRequestResolved` | 同上 |
| `PermissionsRequestApproval` | 已迁入 `ServerRequestResolved` | 同上 |
| `DynamicToolCall` | semantic split | transport 复用 request machinery，但完成语义走 `ItemStarted` / `ItemCompleted` |
| `ChatgptAuthTokensRefresh` | global/threadless | 走外部 auth refresh bridge，不绑定 thread，不适合 resolved thread semantics |
| `ApplyPatchApproval` | legacy holdout | deprecated V1，callback 完成，但不走 V2 resolved path |
| `ExecCommandApproval` | legacy holdout | deprecated V1，callback 完成，但不走 V2 resolved path |

这张表基本就是这次的主结论。

---

## 3. 五个“已迁入 resolved”的 request，有统一模式

这 5 个：

- `CommandExecutionRequestApproval`
- `FileChangeRequestApproval`
- `ToolRequestUserInput`
- `McpServerElicitationRequest`
- `PermissionsRequestApproval`

共同模式都一样：

1. `send_request(...)` 发出去
2. response handler 拿到 `pending_request_id`
3. 先 `resolve_server_request_on_thread_listener(&thread_state, pending_request_id).await`
4. 再把业务结果 `submit(...)` 回 core

这个模式说明：

> **它们的“完成”不只是 callback 结束，而是显式进入 thread listener 排序后的 resolved 语义。**

这才叫真正接进 `ServerRequestResolved`。

而且测试也在验证：

- `serverRequest/resolved` 出现
- 并且顺序早于后续 `turn/completed` 或 item 后续事件

所以这 5 个已经不是 open question 了。

---

## 4. `DynamicToolCall`：不是缺口，是 semantic split

这点上一轮已经单独打过了，这里只收口。

它的形态是：

- `send_request(...)` 发 `item/tool/call`
- 进入 pending request map
- 可 replay / 可 cancel
- 但客户端响应后，不走 `resolve_server_request_on_thread_listener(...)`
- 而是直接 `submit(Op::DynamicToolResponse { ... })`
- 整个对外完成语义由：
  - `ItemStarted`
  - `DynamicToolCall`
  - `DynamicToolCallResponse`
  - `ItemCompleted`
  来承担

所以最准确的一句话还是：

> **它是 transport 上的 request，语义上的 item。**

这类不该再算“没迁完”。

---

## 5. `ChatgptAuthTokensRefresh`：不是 thread request，而是 global bridge request

这个 request 和前面那批 thread request 不是同一个世界。

证据很直接：

- 代码在 `message_processor.rs` 的 `ExternalAuthRefreshBridge`
- 直接 `outgoing.send_request(ServerRequestPayload::ChatgptAuthTokensRefresh(...))`
- 然后本地 timeout / cancel / parse response
- 没有 thread listener，也没有 `thread_state`
- 没有 `resolve_server_request_on_thread_listener(...)`

所以这里不是“漏了 resolved”。

而是：

> **它根本不属于 thread-scoped resolved 语义模型。**

这类应该单独看成 global/client-bridge request。

---

## 6. V1 deprecated 两个 request：本质就是 legacy holdout

这两个：

- `ApplyPatchApproval`
- `ExecCommandApproval`

也会：

- `send_request(...)`
- 等 callback
- deserialize response
- `submit(...)` 回 core

但和 V2 resolved 那 5 个不同，它们的 response handler 里没有：

- `resolve_server_request_on_thread_listener(...)`

而且发送分支本身就被包在：

- `ApiVersion::V1`

下面。

所以它们最准确的归类不是“未知”，而是：

> **deprecated V1 legacy holdout。**

后面要不要清理它们，是演进路线问题；不是当前语义归属不明的问题。

---

## 7. 现在还剩什么 open question

真正还没完全打死的，不再是“当前枚举里谁最可疑”。

因为当前 9 个 shape 已经都能归类。

现在剩的 open question 更像两类：

### 1) future evolution question
比如：
- 未来会不会把 legacy V1 request 全部删掉
- 会不会再把某些 global request 纳入别的统一观察面

### 2) repo drift question
比如：
- 后面如果枚举里新增 request shape，它会落在哪一类

也就是说：

> **当前问题已经不是分类不清，而是未来是否还会改分类。**

---

## 8. 对 guidebook / handoff 的意义

这次收口后，后面正文里可以直接这么写：

- `ServerRequestResolved` 不是“所有 request 完成”的通用代名词
- 它只覆盖一批 **thread-scoped V2 interactive request**
- `DynamicToolCall` 是 semantic split
- `ChatgptAuthTokensRefresh` 是 global bridge request
- `ApplyPatchApproval` / `ExecCommandApproval` 是 legacy holdout

这样写的好处是：

- 不会把 callback 完成和 resolved semantics 混成一件事
- 不会再把 `DynamicToolCall` 误写成未迁移洞
- 也不会把 V1 尾巴误说成“当前主线缺口”

---

## 一句话拍板

> **当前 `ServerRequest` 枚举里的 9 类 request 已经都能归类：5 个 resolved、1 个 semantic split、1 个 global bridge、2 个 legacy holdout。接下来不是继续猜谁最可疑，而是确认未来还有没有新 shape 或删尾巴。**

## 延伸阅读

- [2026-04-12-DynamicToolCall为什么不走ServerRequestResolved.md](2026-04-12-DynamicToolCall为什么不走ServerRequestResolved.md)
- [卷四 03｜`ServerRequestResolved` 到底覆盖了什么控制面语义](../guidebookv2/volume-4/2026-04-12-Codex-卷四-03-ServerRequestResolved-到底覆盖了什么控制面语义-v1.md)
