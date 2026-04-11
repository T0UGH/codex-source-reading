# Codex 源码拆解 31：backend-client 与 model transport 栈

## 这篇看什么

回答：**Codex 最底层的模型/HTTP/WebSocket/SSE 传输栈到底怎么分层？`codex-client`、`codex-api`、`backend-client`、`core::ModelClient` 各自站在哪？**

## 先给结论

这条链现在可以比较完整地说清楚：

1. `codex-client`
   - 最底层 transport substrate
   - 负责 request/response、reqwest transport、retry、custom CA
2. `codex-api`
   - provider-aware API layer
   - 负责 auth/header/provider URL/Responses SSE/WebSocket
3. `core::ModelClient`
   - 运行时 orchestration 层
   - 决定走 HTTP SSE 还是 websocket、怎么注入 provider/auth/telemetry
4. `backend-client`
   - 另一条独立 REST client 线
   - 面向 task/backend API，不是 Responses/model transport 主链

一句话：

> `codex-client` 是底座，`codex-api` 是模型 API 适配层，`ModelClient` 是运行时编排层，`backend-client` 则是另一条后端业务 API 客户端。

## 关键代码路径

- `codex-rs/codex-client/src/transport.rs`
- `codex-rs/codex-client/src/retry.rs`
- `codex-rs/codex-client/src/custom_ca.rs`
- `codex-rs/codex-api/src/provider.rs`
- `codex-rs/codex-api/src/endpoint/session.rs`
- `codex-rs/codex-api/src/endpoint/responses.rs`
- `codex-rs/codex-api/src/sse/responses.rs`
- `codex-rs/codex-api/src/endpoint/responses_websocket.rs`
- `codex-rs/core/src/client.rs`
- `codex-rs/backend-client/src/client.rs`

## 第一层：`codex-client`

这是最底层通用传输层。

它负责：
- `HttpTransport` 抽象
- `ReqwestTransport`
- `Request` / `Response`
- retry policy / backoff
- custom CA

这说明 `codex-client` 的定位很纯：

> 先解决“怎么稳定地发 HTTP 请求和流式请求”。

### retry
`retry.rs` 提供：
- `RetryPolicy`
- `RetryOn`
- `backoff`
- `run_with_retry`

所以重试策略是在底层 transport 层就已经抽象好的。

### custom CA
`custom_ca.rs` 会统一处理：
- `CODEX_CA_CERTIFICATE`
- `SSL_CERT_FILE`

而且 HTTPS 和 websocket TLS 都尽量共享这套 trust 配置。

这点很关键，因为不然 SSE 和 websocket 会出现不同信任根行为。

## 第二层：`codex-api`

这是 provider-aware API layer。

它在底层 transport 之上再包一层：
- provider base_url / query params / headers
- auth 注入
- endpoint-specific request building
- Responses SSE / websocket 协议映射

### `Provider`
`provider.rs` 负责：
- `url_for_path`
- `build_request`
- `websocket_url_for_path`
- `RetryConfig::to_policy`

也就是说，provider 差异主要在这里被标准化。

### `EndpointSession`
这是 common request executor。

它做的事情是：
- 合并 provider headers + extra headers + auth
- 再通过 telemetry wrapper + retry 发送

所以 `EndpointSession` 更像：

> “带 provider/auth/telemetry 语义的统一请求执行器”。

## Responses SSE 的真正实现位置
这个点很容易看错。

虽然 `codex-client` 有个一般性的 SSE helper，
但 Codex 真正的 Responses SSE 解析主路径在：

- `codex-api/src/sse/responses.rs`

这里会：
- 从 header 先拿 rate limit / model etag / server model 等初始元信息
- 用 `eventsource_stream` 解析 event bytes
- 把 `response.completed` / `response.failed` 等事件映射成 typed stream event

也就是说：

> 通用 SSE substrate 在底层，但“Responses 协议理解”在 `codex-api`。

## Responses websocket 是 SSE 的兄弟路径
不是替代掉整个 API 层。

`responses_websocket.rs` 会：
- 复用 provider URL/header/auth 逻辑
- 走 websocket 到 `/responses`
- 同样继承 custom CA

所以现在的关系更像：

- HTTP SSE
- Responses websocket

是同一层级的两条 transport flavor。

## 第三层：`core::ModelClient`

`ModelClient` 才是 runtime 真正拿来用的入口。

它负责：
- provider resolution
- auth provider 组装
- reqwest client 构建
- `ApiResponsesClient` 创建
- telemetry 注入
- websocket vs HTTP SSE 的选择
- fixture mode 特判

换句话说，`ModelClient` 不是简单薄包装，而是：

> 运行时“选哪条模型传输面”的编排层。

### 一个关键点：stream retry budget 在更高层
底层 request retry 在 `codex-client`/`codex-api` 已经有。

但 turn 级别的：
- stream retry budget
- websocket fallback to HTTPS

是在 `core` 更高层做的。

这说明重试是两层：

1. request-level retry
2. turn/session-level retry orchestration

这个分层是合理的。

## `backend-client` 不属于 model transport 主链
这个要明确说。

`backend-client` 是另一条线：
- rate limits
- task list/details
- sibling turns
- config requirements file
- create task

它更像：

> 面向 Codex backend/task API 的 REST client

不是 Responses/model streaming 主链。

所以后面讲“模型传输栈”时，不能把它和 `codex-client/codex-api/ModelClient` 混成一层。

## 当前边界判断

- `codex-client`
  - transport substrate
  - retry
  - custom CA
- `codex-api`
  - provider-aware API layer
  - auth/header/endpoint shaping
  - Responses SSE/WebSocket mapping
- `core::ModelClient`
  - runtime orchestration
  - transport choice / fallback / fixture
- `backend-client`
  - backend/task REST client
  - 非 model transport 主链

## 设计判断

### 1. 这套分层是合理的
如果没有 `codex-api` 这一层，provider/auth/endpoint 细节会直接污染 runtime。

### 2. `backend-client` 单独存在是对的
因为 task/backend API 和 Responses/model API 的生命周期、错误模型、调用习惯都不一样。

### 3. retry 做成两层也对
- 底层管请求稳定性
- 上层管 turn 语义与 transport fallback

## 还值得继续观察的点

1. websocket 与 HTTP SSE 在 provider 差异上的长期演化
2. `backend-client` 是否未来会收敛到更通用的 transport substrate
3. `codex-client::sse` 与 `codex-api::sse::responses` 是否还会进一步统一
