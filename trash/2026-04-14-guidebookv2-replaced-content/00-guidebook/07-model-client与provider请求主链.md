---
title: Codex guidebook 07｜ModelClient 与 provider 请求主链
date: 2026-04-12
status: draft
---

# 07｜ModelClient 与 provider 请求主链

## 先给结论

如果说前面的几章讲清了 Codex 本地 runtime、thread、turn 和执行子系统，那么这一章要补的是另一条主线：

> **模型请求到底是怎么从 Codex session 里发出去的。**

这条链的核心对象不是 transport 本身，而是 `ModelClient`。

它最合理的定位不是“HTTP 客户端”，而是：

> **session-scoped、turn-aware 的模型请求编排层。**

它自己不直接实现底层 HTTP/WebSocket 细节，但负责：

- 持有 session 级身份与 provider 状态
- 在每个 turn 上组请求
- 决定走 WebSocket 还是 HTTP Responses
- 管理 turn-state sticky routing
- 在 WebSocket 不合适时回退到 HTTP
- 把 Codex 自己的 session / turn 语义，挂进 provider 请求头和请求体里

所以这章的重点不是“一个 crate 调另一个 crate”，而是：

> **Codex 如何把自己的 thread/turn 语义，变成一次 provider 可理解的模型请求。**

---

## 1. `ModelClient` 为什么不是普通 client

从名字看，`ModelClient` 很容易被理解成“和模型通信的客户端”。

但源码里的职责其实更重。它持有的不是一批短命请求参数，而是一整套 session 级上下文，比如：

- auth manager
- conversation/thread id
- installation id
- provider info
- session source
- verbosity / compression / timing 之类的行为配置
- session 级的 websocket fallback 状态
- 可复用的 websocket session

这说明它更像：

> **模型访问编排器**

而不是单纯 transport wrapper。

它既要理解 provider 怎么接，又要理解当前这次请求属于哪个 session、哪个 thread、哪种 turn 语义。

---

## 2. 为什么还要单独有 `ModelClientSession`

如果 `ModelClient` 已经持有 session 状态，为什么还要有 `ModelClientSession`？

原因在于 Codex 很明确地区分了两层：

### `ModelClient`
负责 session 级不变量：
- provider / auth / conversation identity
- session 生命周期上的 websocket fallback 策略
- 可跨 turn 复用的一些连接态

### `ModelClientSession`
负责 turn 级不变量：
- 这一次请求的 websocket session
- 上一个响应的 continuation 信息
- 当前 turn 的 `x-codex-turn-state`
- 当前 turn 的 request/response 粘连关系

所以不是一个 client 加点参数，而是：

> **session 级编排对象 + turn 级执行对象**

这也是为什么 Codex 能在一条会话里保持更强的一致性，而不是每次都把模型调用当成完全独立的 HTTP 请求。

---

## 3. 请求主链：从 session 到 provider

把一条典型模型请求压缩一下，可以拆成下面几步。

### 第一步：构建 turn session
`ModelClient::new_session()` 会创建一个新的 `ModelClientSession`。

这一步不是只是 new 一个小对象，而是把：

- 可能可复用的 websocket state
- 新 turn 的 state 容器
- 与上一次响应相关的 continuation 上下文

一起装配进去。

### 第二步：构建 Responses request
接着，系统会在 turn 上构建 `ResponsesApiRequest`。这里会装入：

- model slug
- instructions
- formatted input
- tools
- reasoning 设置
- 输出 schema
- prompt cache key
- 安装信息和 client metadata

也就是说，到这一步，请求体已经不再只是“模型输入”，而是带了明显的 Codex runtime 语义。

### 第三步：构建 request options / headers
再下一步，会构建 `ResponsesOptions` 和额外 headers，用来挂这些东西：

- conversation id
- session source
- beta / feature headers
- turn state
- turn metadata
- subagent / parent-thread / window identity 等扩展信息

所以这条链非常清楚：

> **先组语义请求体，再组 session/turn 级头信息，最后才进入 transport 选择。**

---

## 4. transport 选择：为什么先试 WebSocket，再回退 HTTP

当前这条主链里，真正的 wire API 还是 Responses。但在 Responses 之上，Codex 会优先尝试 WebSocket path。

逻辑大致是：

- provider 支持 Responses
- 如果 websocket 没被 session 禁用，就先尝试 websocket
- 如果 websocket 返回必须 fallback 的信号，就切到 HTTP
- 一旦切到 HTTP，这个 session 后续会记住“不再优先 websocket”

这说明 WebSocket 在这里不是一个附属 feature，而是一个明确参与 session 级决策的 transport 分支。

这也进一步说明：

> `ModelClient` 管的不只是“调用模型”，而是“在当前 session 里，哪条 transport 路最适合、还能不能继续用”。

---

## 5. WebSocket 路径到底多了什么语义

如果 HTTP 也能 stream，为什么还要 WebSocket？

因为在 Codex 这里，WebSocket 不只是传输方式，它还承载了更多 turn/session 语义。

最关键的有三点：

### 1）预热与连接复用
系统可以提前 prewarm websocket，并在 turn 内尽量复用连接。

### 2）sticky routing
服务端可能返回 `x-codex-turn-state`，客户端把它存起来，只在当前 turn 内继续带上。这样可以保持 turn 内路由一致性。

### 3）增量 continuation
如果当前输入只是上一次输入的扩展，并且上一次 response/items 还可复用，那么 websocket 路径可以只发送 delta，并带上 `previous_response_id`。

这些都说明：

> **WebSocket 在 Codex 里不是更快一点的 HTTP，而是更适合承载 turn 连续性的 transport。**

---

## 6. HTTP 路径并不是降级版，它是稳定 fallback 路径

虽然 WebSocket 更强，但 HTTP 不是退而求其次的简陋通道。

HTTP Responses 路径同样是完整主链的一部分：

- 先解析 auth/provider
- 建 `ReqwestTransport`
- 建 request + telemetry
- 构造 `ApiResponsesClient`
- 调 `stream_request(...)`
- 再把 SSE 流解析回 `ResponseEvent`

它的特点是：

- 更稳定、普适
- 能和通用 HTTP transport/retry/compression 更自然地配合
- 适合作为 session 级 fallback 目标

所以更准确地说：

- WebSocket 是高连续性路径
- HTTP Responses 是稳定通用路径

而两者的上层编排，都由 `ModelClient` 统一管理。

---

## 7. `current_client_setup()` 为什么这么关键

在这条链里，最值得注意的一个 seam 是 `current_client_setup()`。

它负责统一做三件事：

- 从 `AuthManager` 里拿到当前 auth
- 把 `ModelProviderInfo` 转成 `codex_api::Provider`
- 把 auth 材料转成 `CoreAuthProvider`

这让模型请求主链有了一个很稳的统一入口：

- prewarm 走这里
- HTTP streaming 走这里
- WebSocket connect 走这里
- compact / realtime / memories 这些旁支请求也走这里

所以这不是一个小工具函数，而是：

> **模型请求世界里的 setup seam。**

它把“配置世界”和“执行世界”接起来了。

---

## 8. `ModelProviderInfo` 到 `Provider`：配置层和执行层为什么要分开

这条链上还有一个经常被忽略但很重要的边界：

- `ModelProviderInfo` 是用户/配置视角
- `codex_api::Provider` 是执行视角

两者不是同一个东西。

前者更偏：
- schema
- 用户配置
- provider 信息描述

后者更偏：
- base URL
- query params
- headers
- retry config
- timeout
- 实际请求执行所需的 provider 细节

所以这一步转换的意义，不只是类型适配，而是：

> **把“人理解的 provider 配置”转换成“系统可执行的 provider 请求参数”。**

这也是整个 transport 栈里很稳的一层边界。

---

## 9. 这一章最重要的判断

读完这章，应该稳定这些结论。

### 判断 1：`ModelClient` 不是 transport 本身
它是 session/turn 级模型请求编排层。

### 判断 2：`ModelClientSession` 是 turn-aware 对象
它持有当前 turn 的 websocket/continuation/state 语义。

### 判断 3：请求主链先组语义，再选 transport
而不是先拿一个 transport 再往里塞业务字段。

### 判断 4：WebSocket 在这里承载 turn 连续性
不是单纯更快的通道。

### 判断 5：HTTP Responses 是稳定 fallback 主链
不是低配版实现。

### 判断 6：`current_client_setup()` 是关键 setup seam
它统一 auth/provider 的进入方式。

### 判断 7：`ModelProviderInfo -> Provider` 是配置层到执行层的转换
不是简单类型重命名。

---

## 10. 下一章该接什么

这一章把“模型请求是怎么出去的”讲清楚了。接下来最自然的问题就是：

> **这条模型 transport 栈里，各个 crate 到底怎么分层？`codex-client`、`codex-api` 和 `backend-client` 为什么不是一回事？**

也就是下一章的主题：

- `codex-client`
- `codex-api`
- `backend-client`
- HTTP/SSE/WebSocket 边界
- 为什么 backend-client 不属于 model inference transport

---

## 相关阅读

- 想继续看 transport substrate / backend 边界：[[08-codex-client-codex-api与backend-client分层]]
- 想看模型请求主链对应的关键函数与调用顺序：[[11-关键函数索引]]、[[12-调用链索引]]

## 关键文件

- `codex-rs/core/src/client.rs`
- `codex-rs/login/src/api_bridge.rs`
- `codex-rs/model-provider-info/src/lib.rs`
- `codex-rs/codex-api/src/common.rs`
- `codex-rs/codex-api/src/provider.rs`
- `codex-rs/codex-api/src/endpoint/responses.rs`
- `codex-rs/codex-api/src/endpoint/responses_websocket.rs`
