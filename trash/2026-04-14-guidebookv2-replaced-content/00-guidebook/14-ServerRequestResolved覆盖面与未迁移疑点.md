---
title: Codex guidebook 14｜ServerRequestResolved 覆盖面与未迁移疑点
date: 2026-04-12
status: draft
purpose: micro-topic
---

# 14｜ServerRequestResolved 覆盖面与未迁移疑点

## 先给结论

这是一个很小，但很值继续打死的点：

> **app-server 里并不是所有 request 都会走 `ServerRequestResolved` 语义。**

继续往下看，情况还能再分成三类：

1. **已经完整迁到 `ServerRequestResolved` 的 V2 thread-scoped request**
2. **本来就不适合走这套语义的 request**
3. **看起来像还没迁完的 request**

而当前最值得记录、也已经进一步收敛的点只有一个：

> **`DynamicToolCall` 虽然是 thread-scoped、可 replay、也属于 V2 风格 request，但现在更适合被理解为：复用了 pending request transport 机制，却故意不走 `ServerRequestResolved`，而是走 item lifecycle。**

这篇要做的不是继续把它当“最可疑未迁移点”，而是把这个边界写死，避免以后再把 transport 复用误判成 protocol 语义未迁完。

---

## 1. 先把两个层次分开

理解这个问题，先要把两种“resolved”分开。

### 第一层：callback map 里的请求完成
所有 app-server request 都会进入 `outgoing_message.rs` 的 callback map。客户端回复或报错后，系统都会：

- 找到对应 request id
- 完成 callback
- 从待处理映射里移除

也就是说，**内部 callback 级的 resolve** 是通用存在的。

### 第二层：`ServerRequestResolved` 协议语义
这不是内部 callback 的同义词，而是：

- 通过 listener command channel 串回 thread listener 上下文
- 形成有序的 resolved 语义通知
- 和 replay / resume / protocol item ordering 一起工作

所以真正的问题不是“这个 request 会不会完成”，而是：

> **它会不会进入 app-server 的 `ServerRequestResolved` 语义面。**

---

## 2. 当前已经完整迁到 `ServerRequestResolved` 的 request

这批是最干净的。

它们有几个共同点：

- 都是 thread-scoped
- 都是 V2 交互请求
- 都会被 pending request replay 机制覆盖
- 都会在 handler 里显式调用 `resolve_server_request_on_thread_listener(...)`

目前能确认的有 5 类：

### 1) `CommandExecutionRequestApproval`
特点：
- thread-scoped
- 可 replay
- resolve 后会显式进入 `ServerRequestResolved`

### 2) `FileChangeRequestApproval`
特点和上面一样，也是标准迁移完成态。

### 3) `ToolRequestUserInput`
特点：
- 用户输入请求
- thread-scoped
- resolve path 已接上

### 4) `McpServerElicitationRequest`
特点：
- MCP elicitation 请求
- resolve path 已接上

### 5) `PermissionsRequestApproval`
特点：
- permissions approval 请求
- resolve path 已接上

这 5 类可以视为：

> **当前 `ServerRequestResolved` 语义已经稳定覆盖的主 request 集。**

---

## 3. 这批 request 为什么说是“完整迁移态”

它们不只是“有 resolved 通知”，而是整套模式都齐了：

1. request 进入 thread-scoped pending map
2. reconnect / resume 时能 replay
3. response/error 先完成 callback
4. 再通过 listener command channel 发 `ResolveServerRequest`
5. 最终在 listener 上下文里发出 `ServerRequestResolved`

这套模式的关键在于：

> **resolved 不是异步随手发，而是和 thread listener 的顺序控制绑定在一起。**

所以这些 request 可以算真正迁到了新的 thread-scoped protocol semantics 里，而不是只在 callback map 层面“完成了”。

---

## 4. 哪些 request 看起来是故意不走这套语义

接下来是第二类：

> **它们不走 `ServerRequestResolved`，但这看起来是合理设计，不像缺口。**

### 1) `ChatgptAuthTokensRefresh`
这是最明显的一类。

它的特征是：
- 不是 thread-scoped request
- `thread_id: None`
- 本身更像全局/连接级刷新请求

而 `ServerRequestResolved` 本来就要求站在 thread 语义里发，所以它不适合这套模型。

因此这一类最稳的判断是：

> **故意不进 `ServerRequestResolved`，不是缺口。**

### 2) `ApplyPatchApproval`（deprecated V1）
### 3) `ExecCommandApproval`（deprecated V1）
这两类也没有走 `ServerRequestResolved`。

但和 `DynamicToolCall` 不同，它们带有明显的 legacy/V1 痕迹。更像是：

- 旧协议请求
- callback 会完成
- 也能被 thread pending map / replay 机制覆盖
- 但没有接上新的 V2-style resolved semantics

这类更像：

> **有意保留的旧语义，而不是当前系统里正在积极推进的主线缺口。**

所以它们应该归入“intentional legacy holdout”，而不是当前最值得追的 bug/缺口。

---

## 5. `DynamicToolCall`：transport 上像 request，语义上像 item

这条线仍然最值得记，但结论已经变了。

`DynamicToolCall` 具备这些特征：

- thread-scoped
- 通过 thread request 机制发送
- 会进入 pending request map
- reconnect / resume 时可以 replay
- 属于明显偏新的 V2 request 世界

这些都没错。

但继续往下追会发现，它真正的闭环不是：

- callback 完成
- `resolve_server_request_on_thread_listener(...)`
- `ServerRequestResolved`

而是：

- `ItemStarted`
- `item/tool/call`
- 客户端返回 `DynamicToolCallResponse`
- app-server 直接 `submit(Op::DynamicToolResponse { ... })`
- core 再发 `DynamicToolCallResponse`
- `ItemCompleted`

也就是说，它虽然借用了 thread request transport，
但产品语义上真正暴露给客户端/线程世界的，是 dynamic tool item 的开始与完成。

所以现在更稳的判断是：

> **`DynamicToolCall` 不是最像“还没迁完”的 request；它更像“实现层复用 request，语义层故意走 item lifecycle”的分叉设计。**

---

## 6. 为什么 `DynamicToolCall` 最容易让人误判成未迁移缺口

因为它在 transport 形态上，确实太像那批已经接进 `ServerRequestResolved` 的 request 了。

和 `ApplyPatchApproval` / `ExecCommandApproval` 不同，`DynamicToolCall`：

- 不是明显 V1 遗留
- 不是 threadless/global request
- 不是天然不适合 thread-resolved model
- 本身已经能 replay
- 也站在 V2 request 世界里

所以只看发送/回放/取消机制，很容易得出：

> **它应该也走 `ServerRequestResolved`。**

但这次继续往下追后，真正更稳的理解是：

> **它只是复用了同一套 transport machinery，并没有选择同一套 protocol completion semantics。**

也因此，后面如果还要继续补 gap，不该再优先追它，而该转去看剩余 legacy request 形态里到底还有哪些是真残留、哪些是有意保留。

---

## 7. 现在最稳的一张覆盖面表

可以把当前状态压成这张表。

### A. 已覆盖 `ServerRequestResolved`
- `CommandExecutionRequestApproval`
- `FileChangeRequestApproval`
- `ToolRequestUserInput`
- `McpServerElicitationRequest`
- `PermissionsRequestApproval`

### B. 看起来故意不覆盖
- `ChatgptAuthTokensRefresh`
  - 因为 threadless/global

### C. 看起来是 legacy holdout
- `ApplyPatchApproval`
- `ExecCommandApproval`

### D. transport 复用但语义不走 resolved 的特殊分支
- `DynamicToolCall`
  - thread-scoped / replayable / callback-completing
  - 但完成语义走 `ItemStarted` / `ItemCompleted` + `DynamicToolCallResponse`
  - 不应再简单归类为“最可疑未迁移点”

这张表已经足够让后续继续读的人少走很多弯路。

---

## 8. 这个问题为什么值得单独成文

因为它正好卡在 app-server 语义化成熟度的边缘上。

如果不把它单独写出来，后面很容易出现两种误判：

### 误判 1
“是不是所有 request 都没有 `ServerRequestResolved`？”

不是。已经有一批明显迁完了。

### 误判 2
“是不是所有不发 `ServerRequestResolved` 的 request 都只是没迁完？”

也不是。至少 `DynamicToolCall` 这个点现在更清楚地说明：

- 有些 request 会复用 pending request transport
- 但故意不复用 resolved completion semantics

所以把它写成一篇微专题，比继续把它塞在 open questions 列表里更值。

---

## 9. 这一篇的最终判断

读完这篇，应该稳定这几个结论。

### 判断 1：callback resolve 和 `ServerRequestResolved` 不是一回事
前者是内部 callback map 完成，后者是 thread-aware protocol semantics。

### 判断 2：已经有 5 类 V2 thread-scoped request 完整迁到了 `ServerRequestResolved`
这不是一个“完全没落地”的机制。

### 判断 3：`ChatgptAuthTokensRefresh` 不进这套语义是合理的
因为它不是 thread-scoped。

### 判断 4：V1 deprecated requests 不进这套语义，更像 intentional legacy holdout
不该优先追。

### 判断 5：`DynamicToolCall` 不该再被写成“最像未迁移完成的缺口”
它更准确的边界是：transport 上复用 thread-scoped request machinery，语义上走 item lifecycle。

---

## 10. 如果继续追，这一刀怎么下

如果未来还要继续补这条线，最值的动作就不该再围着 `DynamicToolCall` 打转了。

更值的是：

> **顺剩余 legacy request 形态继续查，确认还有哪些是真正没迁到 `ServerRequestResolved`，哪些只是像 `DynamicToolCall` 一样在语义上故意走别的完成模型。**

具体建议：
- 对照那 5 类已迁 request 的 resolve path
- 继续查 legacy / edge request 分支
- 区分：
  - intentional semantic split
  - intentional legacy holdout
  - actual incomplete migration

这才是这条线后续最自然的一刀。

---

## 延伸阅读

- [03-boundary-judgments/2026-04-12-DynamicToolCall为什么不走ServerRequestResolved.md](../03-boundary-judgments/2026-04-12-DynamicToolCall为什么不走ServerRequestResolved.md)

## 关键文件

- `codex-rs/app-server/src/outgoing_message.rs`
- `codex-rs/app-server/src/thread_state.rs`
- `codex-rs/app-server/src/codex_message_processor.rs`
- `codex-rs/app-server/src/bespoke_event_handling.rs`
- `codex-rs/app-server/src/dynamic_tools.rs`
- `codex-rs/app-server-protocol/src/protocol/common.rs`
- `codex-rs/app-server-protocol/src/protocol/v2.rs`
