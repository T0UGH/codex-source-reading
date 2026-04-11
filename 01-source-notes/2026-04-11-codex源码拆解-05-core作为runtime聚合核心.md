# Codex 源码拆解 05：core 作为 runtime 聚合核心

## 这篇看什么

回答：**`codex-rs/core` 为什么是整个系统最像“大脑”的地方？它到底聚合了哪些能力？**

## 先给结论

`codex-core` 不是一个普通公共库，而是 Codex 的 **runtime aggregation layer**。

它同时汇聚：
- thread/session lifecycle
- config / config loader
- MCP 管理
- skills
- plugins
- rollout / session persistence bridge
- sandbox / shell / exec
- model client / prompt / event mapping

简单说：

> `core` 不是一个小而纯的 domain crate，而是 Codex runtime 的系统中枢。

## 关键代码路径

- `codex-rs/core/src/lib.rs`
- `codex-rs/core/src/thread_manager.rs`
- `codex-rs/core/src/codex_thread.rs`
- `codex-rs/core/src/codex.rs`
- `codex-rs/core/src/exec.rs`
- `codex-rs/core/src/tools/handlers/unified_exec.rs`
- `codex-rs/core/src/unified_exec/process_manager.rs`

## 关键证据

### 1. `core/src/lib.rs` 的导出面极宽
从导出可以看出，`core` 不是单一职责：
- `ThreadManager`
- `CodexThread`
- `McpManager`
- config / config_loader
- skills / plugins
- rollout helpers
- sandboxing
- shell / spawn
- state DB bridge
- review / web search / tools

这本身就是“聚合核心”的直接证据。

### 2. `ThreadManager` 管全局 runtime 资源
`thread_manager.rs` 里能看到 `ThreadManagerState` 维护：
- model/auth/env
- MCP manager
- skills manager
- plugins
- thread map

这说明 `ThreadManager` 是全局线程与共享服务的协调者。

### 3. `CodexThread` 是线程级交互面
`codex_thread.rs` 暴露：
- submit
- next_event
- steer_input
- rollout access
- config snapshot
- MCP calls

这说明 `CodexThread` 是 thread-facing API，不是最底层 loop，但也不是 UI 层。

### 4. `Codex` 才是更底层的 session/turn loop 承载者
`codex.rs` 里可以看到：
- submission 通过 channel 进入内部流程
- event 通过 channel 输出
- turn 内部会构造 prompt / tools / sampling request / tool runtime

这里才最接近 agent loop 本身。

### 5. exec/tool execution 也收进了 core
这里有两层：
- `core/src/exec.rs`：较低层的 sandboxed command execution plumbing
- `core/src/tools/handlers/unified_exec.rs` + `unified_exec/process_manager.rs`：更高层的 persistent/background/TTY-aware exec runtime

说明执行工具运行时也被纳入 `core` 主体，而不是挂在外围。

## 初步结构判断

### core 内部可以粗分为几类
1. **生命周期层**：thread/session/turn
2. **能力接入层**：MCP、skills、plugins、hooks
3. **执行层**：shell、sandbox、unified exec
4. **持久化桥接层**：rollout、state DB bridge
5. **模型交互层**：client、prompt、event mapping

## 设计判断

### 1. `core` 是“中枢”，不是“纯领域层”
从工程审美看它有点重，但从产品落地看很合理：
Codex 需要一个地方把线程、工具、模型、状态、持久化、扩展点真正接起来。

### 2. `core` 的存在说明 Codex 明显不是一堆壳的拼装
它有一个明确的运行时中心，而不是 UI 直接拼工具。

### 3. 这也解释了为什么 TUI / exec / app-server 都能共用主体
因为它们上面站着一个足够厚的 runtime core。

## 当前边界判断

- `core`：runtime 聚合核心
- `ThreadManager`：全局线程与共享服务协调者
- `CodexThread`：线程级 API 面
- `Codex`：更底层的 session/turn loop 容器

## 还没确认的点

1. `Codex::spawn` 内部完整 loop 还没单独展开。
2. `core::exec.rs` 与 `unified_exec/*` 的最终分工还需要单独写一篇。
3. `core` 和 `app-server` 的职责边界虽然大体清楚，但中间桥接面还值得再拆。

## 下一步建议

继续写：
- `interactive TUI 主链草图`
- `exec 非交互主链草图`
- `core 里的执行能力：legacy exec vs unified exec`
