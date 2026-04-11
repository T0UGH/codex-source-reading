# Codex 源码拆解 25：exec-server 体系

## 这篇看什么

回答：**exec-server 到底是什么？它和 `unified_exec`、`EnvironmentManager`、本地/远端执行是什么关系？**

## 先给结论

`exec-server` 不是一个小 helper，而是一个完整的：

> **进程执行 + 文件系统 RPC 服务**

它的核心分层是：

1. server 侧
   - JSON-RPC websocket 服务
   - process handler / file system handler
   - session registry
2. client 侧
   - `ExecServerClient`
   - request/notification 路由
   - remote process/session handle
3. environment 抽象层
   - `EnvironmentManager`
   - 统一本地/远端执行与文件系统

一句话：

> core 不直接依赖“远端 websocket”，而是依赖 `Environment`；exec-server 只是 remote/local environment 的具体实现后端之一。

## 关键代码路径

- `codex-rs/exec-server/src/lib.rs`
- `codex-rs/exec-server/src/client.rs`
- `codex-rs/exec-server/src/server.rs`
- `codex-rs/exec-server/src/server/session_registry.rs`
- `codex-rs/exec-server/src/local_process.rs`
- `codex-rs/exec-server/src/remote_process.rs`
- `codex-rs/exec-server/src/environment.rs`
- `codex-rs/core/src/unified_exec/process_manager.rs`

## transport/protocol 视角

生产形态主要是：
- websocket 上跑 JSON-RPC

协议方法包括：
- `initialize`
- `process/start`
- `process/read`
- `process/write`
- `process/terminate`
- `fs/readFile` 等

而 process 的输出/退出又会通过 notification 告知 client。

所以 exec-server 本质上是：

> 一个 process/filesystem remote runtime service。

## server 端的关键点

### 1. session registry 在 server 端
`session_registry.rs` 负责：
- `session_id -> SessionEntry`
- attach/detach
- resumable session
- detach TTL 过期清理

这说明 exec-server 的“会话语义”不是 client 本地虚拟出来的，而是 server 真正维护的。

### 2. server 的执行后端始终是 local
这点很关键。

server 端 `ProcessHandler` / `FileSystemHandler` 包的是：
- `LocalProcess`
- `LocalFileSystem`

所以所谓 remote exec 的真实含义是：

> 你连到一台远端机器上的本地 exec-server，让它替你本地执行。

不是 server 再递归连另一个 remote backend。

## client 端的关键点

### 1. client 有自己的 notification demux
client 也会维护：
- `process_id -> SessionState`

但它的职责不是保存真正 session 真相，而是：
- 把 connection-global notifications 路由到正确 remote process handle
- 唤醒等待 `process/read` 的 reader

所以：
- server session registry = 真正的 session state
- client session map = 本地事件路由辅助

### 2. notification 不是输出正文
这一点很容易误解。

notification 主要是：
- output available
- exited
- closed

真正的输出读取仍然走：
- `process/read`

这说明 exec-server 没把 websocket notification 设计成完整字节流通道，而是：

> notification 做 wakeup，正文用显式 read 拉。

## local vs remote process 抽象
`ExecBackend` / `ExecProcess` 是统一接口：

### LocalProcess
- 本地用 PTY/pipe 启动执行

### RemoteProcess
- 底层调 `ExecServerClient`
- 对外仍长得像 `ExecProcess`

这一步抽象很重要，因为它让 core 高层不用关心：
- 这个 process 是本地 spawn 的
- 还是远端 RPC 出来的

## local vs remote filesystem 抽象
同理也有：
- `LocalFileSystem`
- `RemoteFileSystem`

远端文件系统操作本质上是把 trait 调用翻译成 exec-server RPC。

所以 `Environment` 不是只决定“命令在哪跑”，也决定：

> 文件系统视角到底是本地还是远端。

## `EnvironmentManager` 的真实作用
`EnvironmentManager` 是 session-scoped selector/cacher：

- 没配置 `CODEX_EXEC_SERVER_URL` → local environment
- `none` → disabled environment
- websocket URL → remote environment

它会把选出来的 `Environment` 缓存下来。

也就是说，core 上层真正依赖的是：
- `EnvironmentManager`
- `Environment`

而不是直接依赖 exec-server transport。

## 和 `unified_exec` 的关系
`unified_exec` 并不直接“知道 websocket”。

它只看：
- `turn.environment`
- `environment.get_exec_backend()`

如果 remote：
- 走 exec-server backend

如果 local：
- 走本地 PTY/pipes

而且 remote 模式下：
- 不支持 inherited FDs
- 会跳过一些本地 shell snapshot 包装
- 某些本地特化 backend（如 zsh-fork）也不支持

所以可以把它们的关系说成：

- `unified_exec` = 执行产品层
- `Environment` = 本地/远端执行抽象层
- `exec-server` = remote/local service implementation

## 当前边界判断

- `exec-server`：process + filesystem RPC 服务
- `SessionRegistry`：server-side session 真相源
- `ExecServerClient`：remote RPC client
- `EnvironmentManager`：session-scoped environment selector
- `Environment`：exec + fs 的统一视图
- `unified_exec`：上层消费 environment 的执行产品

## 设计判断

### 1. 这套抽象是对的
如果没有 `Environment` 这一层，core 高层会被 websocket/process/fs 细节污染得很严重。

### 2. exec-server 是“远端机上的本地执行器”
这个判断非常关键，避免把它脑补成多级代理系统。

### 3. 文件系统也被一起抽象进 environment，是正确决定
否则 remote exec 很快就会出现 cwd/path/fs 视角错位。

## 还值得继续观察的点

1. exec-server 的 auth/TLS/部署边界还没继续深挖
2. 谁负责在真实系统里启动/发现远端 exec-server，还可再看上层 orchestrator
3. local/remote process read/write 通知策略还有没有更细的性能权衡
