---
title: Codex guidebook 08｜codex-client、codex-api 与 backend-client 分层
date: 2026-04-12
status: draft
---

# 08｜codex-client、codex-api 与 backend-client 分层

## 先给结论

Codex 里至少有两套完全不同的“client 栈”：

1. **模型请求栈**  
   `core::ModelClient -> codex-api -> codex-client`

2. **backend 业务请求栈**  
   `app-server / cloud-* -> backend-client`

如果把这两条线混在一起，就会误以为：

- backend-client 是某种 provider transport
- codex-client/codex-api 只是 backend-client 的另一层包装

这两个判断都不对。

更准确的说法是：

> **`codex-client` 是通用 HTTP transport substrate，`codex-api` 是模型 wire API adapter，`backend-client` 是另一套面向 Codex backend 业务资源的 reqwest 客户端。**

也就是说：

- 模型栈负责把 inference request 送到 provider
- backend-client 负责访问任务、限额、配置文件等 backend 资源

它们服务的是不同对象、不同协议面、不同语义目标。

---

## 1. 第一层：`codex-client` 是原始 transport substrate

先看最底层。

`codex-client` 做的事情非常克制。它不懂 Responses，不懂 compact，也不懂 memories，更不懂任务列表或云端 requirements。它主要负责的是这些：

- 通用 request/response 类型
- `HttpTransport` 抽象
- `ReqwestTransport` 实现
- retry policy
- compression
- 一些 trace header 注入相关 HTTP 包装

所以这层最合理的定位不是“API client”，而是：

> **通用 HTTP transport substrate。**

你可以把它理解成模型栈里的原始道路层：

- 车怎么跑
- 重试怎么做
- 流式响应怎么返回
- 压缩和 header 怎么处理

但它不决定“车上装的是什么业务”。

---

## 2. 第二层：`codex-api` 是 model-wire adapter

如果 `codex-client` 是道路层，那么 `codex-api` 就是把 Codex 的模型意图翻译成 provider wire API 的那层。

它负责的核心事情包括：

- 定义 `Provider` 和 auth abstraction
- 定义 Responses/compact/memories/realtime 等 endpoint client
- 组织请求路径、请求头和请求体
- 把 SSE / WebSocket / endpoint-specific 事件解析成统一语义流
- 把 provider retry 配置映射到底层 transport retry

所以 `codex-api` 最准确的定位是：

> **模型 API 语义适配层。**

这层已经不是纯 transport，但也还不是 session 编排层。它关心的是：

- 这个 provider 的 endpoint 怎么调
- 这类请求的 body 长什么样
- response event 怎么 parse
- auth 怎么附着

所以它站在 transport 之上、core 之下。

---

## 3. 第三层：`core::ModelClient` 是编排层，不是 API adapter

在上一章里已经讲过，`ModelClient` 自己也不是 transport。

放到这条分层里，它的位置更准确：

> **建立在 `codex-api` 之上的 session/turn 编排层。**

这条链现在就清楚了：

- `codex-client`：通用 HTTP transport substrate
- `codex-api`：provider/model endpoint 适配层
- `core::ModelClient`：session/turn orchestration layer

所以如果要一句话概括模型栈的分工，就是：

> **底层负责把请求发出去，中层负责把 provider API 讲明白，上层负责把 Codex 的会话语义装进请求。**

---

## 4. `EndpointSession` 为什么是 `codex-api` 里的关键 seam

`codex-api` 里最值得注意的结构之一，是 `EndpointSession<T, A>` 这一层。

它的价值在于：

- transport、provider、auth 可以被统一持有
- endpoint client 之间的大部分公共流程可以复用
- 每个 endpoint 只需要关注：
  - path
  - body
  - 额外 headers
  - response parsing
  - stream parsing

也就是说，`codex-api` 没有把每个 endpoint 都写成一套从头到尾完全独立的 client，而是明确抽出一个公共执行 seam。

这使得这层的边界很清晰：

> **它不是“乱七八糟的 endpoint 工具集合”，而是一套有稳定公共骨架的 model-wire adapter。**

---

## 5. HTTP 和 WebSocket 在这套分层里并不在同一层实现

这是一个很容易误读的地方。

很多人会直觉地以为：既然有 `codex-client::HttpTransport`，那 WebSocket 大概也会在同一层抽象里。但实际不是这样。

### HTTP/SSE
这条线主要在：
- `codex-client`
- `codex-api` 的 endpoint clients

这里的责任边界很清楚：

- `codex-client` 负责 HTTP request/stream
- `codex-api` 负责把字节流 parse 成 Responses event

### WebSocket
Responses WebSocket 主要直接在 `codex-api` 里实现。

也就是说：

- `codex-client` 没有抽出一个通用 websocket transport substrate
- WebSocket 更像 model-wire adapter 这层的一种专门实现

这说明一件事：

> **在 Codex 当前架构里，HTTP transport 被抽象成通用 substrate，而 WebSocket 仍然被视为更贴近模型协议层的特殊路径。**

这不是小细节，而是一个很真实的架构边界。

---

## 6. model transport 和 backend-client 到底差在哪

这是这章最重要的问题。

### model transport 栈解决什么
模型栈解决的是：

- 把 inference request 发给 provider
- 处理 Responses/compact/memories/realtime 这类模型相关 endpoint
- 处理 SSE / WebSocket / response event parsing
- 处理 turn-state sticky routing
- 处理 incremental continuation
- 处理 provider auth / provider retry / provider metadata

### backend-client 解决什么
`backend-client` 解决的是另一类事情：

- rate limits
- tasks
- sibling turns
- managed requirements file
- 一些 Codex backend / ChatGPT backend 业务资源访问

也就是说，它访问的是：

- `/api/codex/...`
- `/wham/...`
- 这类业务 endpoint

而不是 provider inference endpoint。

所以最稳的说法是：

> **model transport 是模型推理请求栈，backend-client 是业务资源请求栈。**

它们不是同一个系统里的上下层，而是两条并列 client 栈。

---

## 7. 为什么 `backend-client` 不属于 model inference transport

这件事可以从四个角度看清楚。

### 1）访问目标不同
- 模型栈访问 provider model API
- backend-client 访问 Codex backend / ChatGPT backend 业务 API

### 2）抽象目标不同
- 模型栈处理 inference request / event stream
- backend-client 处理 typed resource fetch/create/list 调用

### 3）transport 需求不同
模型栈需要：
- SSE
- WebSocket
- sticky routing
- incremental continuation
- response event parsing

backend-client 需要的只是普通 reqwest request/response。

### 4）auth 语义不同
- 模型栈围绕 provider auth abstraction
- backend-client 更直接构造 bearer/account 类业务请求头

所以把 backend-client 看成 transport 变体，会从根上看错。

---

## 8. 现在最合理的一张 client 分层图

可以把 Codex 的 client 相关系统压缩成下面这张图。

### 模型请求栈
- `core::ModelClient`
  - session / turn orchestration
- `codex-api`
  - model/provider wire API adapter
- `codex-client`
  - generic HTTP transport substrate

### backend 业务请求栈
- `backend-client`
  - tasks / rate limits / managed files / backend resources

### 关键判断
- 这两条线都叫“client”
- 但它们根本不是同一条架构线的不同层
- 它们是服务不同对象的两套 client 栈

---

## 9. 这一章最重要的判断

读完这章，应该稳定这些结论。

### 判断 1：`codex-client` 是通用 HTTP substrate
它不懂 provider 业务语义。

### 判断 2：`codex-api` 是 model-wire adapter
它负责 endpoint、auth、response parsing、provider-specific 细节。

### 判断 3：`ModelClient` 是编排层
它不直接实现 transport，而是建立在 `codex-api` 之上。

### 判断 4：HTTP 和 WebSocket 没被抽成同一 transport substrate
HTTP 更底层，WebSocket 更靠近 model-wire adapter 层。

### 判断 5：`backend-client` 是另一套业务资源 client
不是 model inference transport 的一部分。

### 判断 6：model transport 和 backend-client 是并列 client 栈
不是同一条链上的上下层。

---

## 10. 下一章该接什么

model transport 和 backend-client 分清以后，接下来最自然的深挖就不再是“模型请求怎么出去”，而是：

> **系统内部的审查与协作机制是怎么挂到 runtime 上的？**

更具体地说，就是：

- `/review` 和 guardian 到底怎么分工
- guardian 为什么不是 review 的另一个名字
- 审批路由、子会话、app-server 投影是怎么接起来的

也就是下一篇深挖的主题：**review 工作流与 guardian 审查基础设施。**

---

## 相关阅读

- 想先看 transport 栈的上层编排怎么装请求：[[07-model-client与provider请求主链]]
- 想直接找 transport / backend 边界的关键函数与链路：[[11-关键函数索引]]、[[12-调用链索引]]

## 关键文件

- `codex-rs/codex-client/src/lib.rs`
- `codex-rs/codex-client/src/request.rs`
- `codex-rs/codex-client/src/transport.rs`
- `codex-rs/codex-client/src/retry.rs`
- `codex-rs/codex-api/src/lib.rs`
- `codex-rs/codex-api/src/provider.rs`
- `codex-rs/codex-api/src/endpoint/session.rs`
- `codex-rs/codex-api/src/endpoint/responses.rs`
- `codex-rs/codex-api/src/endpoint/responses_websocket.rs`
- `codex-rs/backend-client/src/client.rs`
- `codex-rs/app-server/src/codex_message_processor.rs`
- `codex-rs/cloud-requirements/src/lib.rs`
