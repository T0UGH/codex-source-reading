# Codex 源码拆解 07：MCP、hooks、skills 的接入层

## 这篇看什么

回答：**Codex 的外部能力到底怎么接进 runtime？MCP、hooks、skills 分别在系统中处在哪一层？**

## 先给结论

Codex 在这里不是把所有扩展机制混成一团，而是做了明确分层：

- `rmcp-client`：外部 MCP server 的 transport/client 适配层
- `codex-mcp`：MCP runtime integration 层
- `mcp-server`：Codex 作为 MCP server 的暴露面
- `skills`：系统 skill 资产与 skill runtime
- `hooks`：生命周期/事件触发的 hook 机制
- `core`：把上面这些能力真正纳入线程/turn runtime

也就是说：
> MCP、hooks、skills 不是同类东西，它们只是都被 runtime 吸纳进来。

## 关键代码路径

- `codex-rs/codex-mcp/src/lib.rs`
- `codex-rs/codex-mcp/src/mcp/mod.rs`
- `codex-rs/codex-mcp/src/mcp_connection_manager.rs`
- `codex-rs/rmcp-client/src/lib.rs`
- `codex-rs/mcp-server/src/lib.rs`
- `codex-rs/hooks/src/registry.rs`
- `codex-rs/hooks/src/engine/discovery.rs`
- `codex-rs/hooks/src/engine/dispatcher.rs`
- `codex-rs/core/src/hook_runtime.rs`
- `codex-rs/skills/src/lib.rs`
- `codex-rs/core/src/skills.rs`
- `codex-rs/core/src/thread_manager.rs`

## MCP 分层判断

### 1. `rmcp-client` 不是能力层，是 transport/client 层
它负责：
- stdio / HTTP 等 transport
- OAuth / auth 状态
- 外部 MCP server 通信

这说明它本质上是：
> “如何连上别人的 MCP server”

### 2. `codex-mcp` 才是 Codex 内部的 MCP integration 层
`McpConnectionManager` 是关键对象。
它负责：
- 管多个 MCP server 连接
- 并行启动 server
- 聚合工具
- 规范化工具命名
- 路由 tool call 到正确 server
- 处理 elicitation / approval policy
- 推送 sandbox state

这说明 `codex-mcp` 不是 transport，而是：
> “MCP 如何在 Codex runtime 内成为可用能力”

### 3. `mcp-server` 是反向暴露面
它不是 Codex 使用外部 MCP 的路径，而是：
> Codex 自己作为一个 MCP server 对外提供能力。

这和 `codex-mcp` 完全不是一回事。

## hooks 判断

### 1. hooks 不是技能，也不是 MCP
hooks 的职责是：
- 发现 hook 配置
- 匹配生命周期事件
- 调度并执行 handler
- 把 hook started/completed 等事件接回 runtime

### 2. hook 系统现在是部分实现，不是完全成熟的大一统扩展点
源码里能看到 discovery 明确跳过：
- async hooks
- prompt hooks
- agent hooks

这说明当前 hook 能力面是：
- 有正式 runtime 接入
- 但还不是“什么 hook 都全开”的最终形态

### 3. `core/src/hook_runtime.rs` 才说明 hooks 真的进 runtime 了
也就是说，hooks 不是 UI 小技巧，而是已被接入 session/turn 生命周期的 runtime 机制。

## skills 判断

### 1. `codex-skills` 更偏 skill 资产/安装层
它负责 embedded/bundled skill 资产和本地落盘。

### 2. 真正 runtime 里的 skill 管理主要在 `core`
`core/src/skills.rs` 和 `ThreadManager` 里的 `SkillsManager` 才是关键。

这说明技能不是单纯静态文件，而是要经过 runtime 加载、依赖处理、注入管理。

### 3. skills 与 MCP 有交点，但不是同一层
源码里有专门的 MCP skill dependency 路径，说明：
- skills 可能依赖 MCP
- 但二者仍保持独立抽象

## 设计判断

### 1. MCP / hooks / skills 是 3 条不同能力接入线
- MCP：外部工具能力协议
- hooks：生命周期事件注入
- skills：方法与能力包裹层

### 2. `core` 才是它们真正汇流的地方
这进一步证明 `core` 是 runtime 聚合核心。

### 3. Codex 明显在把外部能力系统工程化
不是简单加几个扩展点，而是每类能力都有自己的正式位置。

## 当前边界判断

- `rmcp-client`：MCP transport/client adapter
- `codex-mcp`：MCP runtime integration
- `mcp-server`：Codex 作为 MCP server 的暴露面
- `hooks`：生命周期事件处理机制
- `skills`：skill 资产与 runtime skill manager
- `core`：这些能力的汇流层

## 还没确认的点

1. `mcp-server` 和 `app-server` 的产品边界还值得单独写一篇。
2. skill ↔ MCP 依赖流还没细看。
3. hooks 当前部分未实现能力的 roadmap 和实际启用范围，还没做更深判断。

## 下一步建议

继续写：
- `sandbox / execpolicy / approval` 三者关系
- `app-server 与 embedding 架构`
- `MCP 与 app-server 的关系`
