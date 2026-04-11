# Codex 源码拆解 26：rmcp-client 与 OAuth 链路

## 这篇看什么

回答：**Codex 的 MCP client 侧到底怎么做 transport、恢复、OAuth 和 token 持久化？**

## 先给结论

`rmcp-client` 本质上是：

> 对上游 rmcp SDK 的一层 transport/auth/recovery 包装器

当前可见的重点不是“协议再发明”，而是三件事：

1. transport recipe
   - stdio
   - streamable HTTP
2. OAuth
   - discovery
   - login
   - token persistence/refresh
3. recovery
   - 目前主要针对 streamable HTTP session expiry 404

一句话：

> MCP client 侧最重要的价值，不是 call_tool 本身，而是把 transport/auth/recovery 收敛成一层可用的客户端壳。

## 关键代码路径

- `codex-rs/rmcp-client/src/rmcp_client.rs`
- `codex-rs/rmcp-client/src/auth_status.rs`
- `codex-rs/rmcp-client/src/perform_oauth_login.rs`
- `codex-rs/rmcp-client/src/oauth.rs`
- `codex-rs/rmcp-client/src/program_resolver.rs`
- `codex-rs/codex-mcp/src/mcp_connection_manager.rs`
- `codex-rs/codex-mcp/src/mcp/auth.rs`

## transport：这里只有两条正式路
当前 recipe 很明确：

- `Stdio`
- `StreamableHttp`

这点值得强调，因为很多人会脑补出一堆 HTTP 变体。

从当前代码看，网络路径重点就是 streamable HTTP。

## recovery：当前不是通用 reconnect 框架
恢复逻辑目前最明确的是：

- 如果 streamable HTTP 因为 `SessionExpired404` 失败
- client 会用保存的 transport recipe + initialize context
- 重建 transport
- 再 retry 一次

重要的是：
- 这不是对所有失败都重试
- 401 并不会走同样恢复
- stdio 也没看到同等级的 reconnect 机制

所以正确理解应该是：

> 现在的 recovery 是一条**针对 streamable HTTP session 过期的定向补丁链**，不是全局自愈框架。

## OAuth status 判断链
当前 auth status 大致按这个顺序：

1. 有显式 bearer token env
2. 有 Authorization header
3. 有已存 OAuth token
4. OAuth discovery 成功但没登录
5. 否则 unsupported

这个顺序很合理：
- 先看最明确的静态凭证
- 再看持久化 OAuth
- 最后才说“支持登录但还没登”

## OAuth discovery
`auth_status.rs` 会做 RFC 8414 风格的 well-known metadata 探测。

这说明当前做法不是写死某个服务商，而是偏规范发现。

所以 rmcp-client 在 auth 这块是有“协议意识”的，不只是硬编码几个 URL。

## OAuth login 不是 device code，而是本地浏览器回调流
`perform_oauth_login.rs` 这条链是：

- 起本地 callback server
- 启动 `OAuthState`
- 打开浏览器或返回 URL
- 收 callback
- 换 code
- 存 token

也就是说，当前 OAuth login 重点是：

> local browser callback flow

不是 device-code first。

## token persistence / refresh
`OAuthPersistor` 是这里最有价值的东西之一：

- token 可持久化到：
  - file
  - keyring
  - auto
- fallback file 是 `.credentials.json`
- 调用前会看是否接近过期并 refresh
- 调用后会把更新过的 credential 再持久化

所以 rmcp-client 不只是“把 token 存起来”，而是：

> 持久化层 + 生命周期层 + 调用前刷新层

三者合一。

## 一个关键细节：discovery 失效时会 fallback 到 bearer token
如果以前存过 OAuth token，后来 provider 不再支持 discovery：
- 代码可能直接退回 bearer token 使用路径

这说明 auth 设计很务实：
- discovery 是理想路径
- 不是唯一活路

## stdio transport 的特殊点：program resolution
这块只对 stdio 有意义。

在 Unix：
- 基本依赖系统/shebang

在 Windows：
- 需要更明确地 resolve `.cmd` / `.bat` / `npx` 之类

所以 `program_resolver` 的价值是：

> 把 stdio MCP server 启动的跨平台坑填平。

## core 怎么用这套 client
core 并不直接到处 new `RmcpClient`。

主消费方是：
- `codex-mcp::McpConnectionManager`

它负责：
- 先算 auth status
- 为每个 server 建 client
- initialize
- list tools/resources
- 后续 call_tool

所以可以这样理解：

- `rmcp-client` = transport/auth/recovery 壳
- `codex-mcp` = Codex 自己的 MCP runtime integration
- core = 再通过 `McpConnectionManager` 用它

## 当前边界判断

- `rmcp-client`：MCP client transport/auth/recovery 封装层
- `OAuthPersistor`：token 生命周期管理层
- `perform_oauth_login`：本地浏览器回调登录链
- `McpConnectionManager`：Codex 内部对多 MCP server 的运行时编排层

## 设计判断

### 1. 这层的价值主要在“客户端工程性”而不是协议创新
这是成熟系统该有的样子。

### 2. recovery 目前是局部的，不要高估
如果后面真要做更强鲁棒性，这里还有扩展空间。

### 3. token persistence/refresh 做成独立层是对的
不然 auth 逻辑会散落到每个 MCP client call site。

## 还值得继续观察的点

1. 是否会补更通用的 stdio reconnect / transport recovery
2. app-server/CLI 上层如何进一步包装 OAuth UX
3. streamable HTTP 为什么成为当前唯一网络 transport 主线，后续是否还会扩展
