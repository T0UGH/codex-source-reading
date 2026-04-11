# Codex 源码拆解 23：remote app-server 与 websocket 链路

## 这篇看什么

回答：**Codex 的 remote app-server 是怎么连起来的？stdio / websocket / in-process 各自边界在哪里？TUI 远程模式到底走哪条链？**

## 先给结论

远程 app-server 主链已经很清楚：

1. TUI 选择 `AppServerTarget::Remote`
2. 调 `connect_remote_app_server()`
3. 底层走 `RemoteAppServerClient::connect()`
4. 通过 websocket 跑 JSON-RPC
5. server 侧仍统一进 `MessageProcessor`

一句话：

> remote app-server 不是另一套协议，只是把同一个 app-server contract 放到 websocket transport 上。

## 关键代码路径

- `codex-rs/tui/src/lib.rs`
- `codex-rs/app-server-client/src/remote.rs`
- `codex-rs/app-server/src/main.rs`
- `codex-rs/app-server/src/lib.rs`
- `codex-rs/app-server/src/transport/websocket.rs`
- `codex-rs/app-server/src/transport/stdio.rs`
- `codex-rs/app-server/src/transport/auth.rs`
- `codex-rs/app-server/src/in_process.rs`

## 三种 transport 边界

### 1. stdio
- stdin/stdout 上传 JSON-RPC
- 更像本地 embedding / subprocess 模式

### 2. websocket
- text frame 里跑 JSON-RPC
- 这是 remote app-server 主路径

### 3. in-process
- 没有 socket/process 边界
- 但仍保留 app-server request/notification/event 语义

这里的关键判断是：

**OpenAI 在努力统一 contract，而不是统一 transport。**

## server 启动主链

CLI `main`
→ `run_main_with_transport()`
→ 按配置启 stdio 或 websocket acceptor
→ 所有 incoming message 继续汇到同一个 `MessageProcessor`
→ outgoing 走统一 envelope router

这说明：

- transport 层只是接入差异
- 真正的 server 语义统一在 `MessageProcessor`

## remote client 主链

`RemoteAppServerClient::connect()` 会：

- 建 websocket connection
- 做 initialize handshake
- 起 worker task
- 在 worker 里 multiplex：
  - caller commands
  - websocket inbound frames

所以 remote client 本质上是：

> 一个 websocket 上的 app-server client facade。

不是“浏览器前端”，也不是“简化代理”。

## TUI 的 remote 模式判断

TUI 侧已经很明确：

- embedded / remote 是一等分支
- `AppServerSession` 会统一这两条路径

这意味着 remote 不是临时 debug 功能，而是正式控制面能力。

## websocket auth 的真实角色

remote websocket auth 主要是：

- `Authorization: Bearer <token>` on upgrade

而且 client 侧还限制：
- 只有 `wss://`
- 或 loopback `ws://`

才允许发 auth token。

这个约束很合理，避免把 bearer token 随便明文扔到不可信链路上。

## server 侧 upgrade auth ≠ ChatGPT account auth

这是一个很重要的区分：

### websocket upgrade auth
- 保护 app-server websocket listener
- 可以是 capability token hash
- 也可以是 JWT bearer token（shared secret）

### ChatGPT/account auth refresh
- 属于 app-server protocol 内部 server request
- 不是 websocket upgrade 那一层

所以两套 auth 语义不是一个东西：

- transport auth
- product/account auth

必须分开看。

## in-process 也坚持协议模型的意义

`in_process.rs` 和 client facade 明确表明：

- 即使同进程
- 也尽量保留 app-server protocol semantics

这背后的设计价值很大：

- remote / embedded / local UI 不至于彻底分叉
- 上层更容易在 transport 间切换
- contract 能稳定下来

这也是为什么现在 TUI 能同时支持 embedded / remote，而没有彻底写成两套 UI backend。

## 当前边界判断

- stdio：本地 subprocess transport
- websocket：remote app-server transport
- in-process：同进程 transport，但仍保留 app-server contract
- `MessageProcessor`：真正统一的 server 语义层
- `RemoteAppServerClient`：remote websocket client facade
- TUI：通过 `AppServerSession` 抽象 embedded / remote

## 设计判断

### 1. app-server contract 是一等公民，transport 是二等差异
这个判断越来越稳。

### 2. remote 模式已经不是附属能力，而是正式产品面
至少从代码结构看是这样。

### 3. transport auth 和 product/account auth 的分层做得是对的
不然远程模式会很快变成一团 auth spaghetti。

## 还值得继续观察的点

1. chatgpt auth refresh 在 TUI 侧的完整 UX 还可以继续追
2. `remote_control` 传输面还没展开，可能是另一条控制面线
3. websocket 之外是否还会出现更正式的 HTTP/streaming transport，当前源码还没看到
