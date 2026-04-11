# Codex 源码拆解 13：mcp-server 与 app-server 的产品边界

## 这篇看什么

回答：**`mcp-server` 和 `app-server` 都能驱动 Codex，为什么它们不是同一个东西？**

## 先给结论

两者都复用 `core::ThreadManager`，但产品角色完全不同：

1. `app-server`
   - 是 **Codex 自己的控制平面 / JSON-RPC 服务层**
   - 面向 TUI、SDK、embedding client
2. `mcp-server`
   - 是 **把 Codex 包成一个 MCP 工具**
   - 面向外部 MCP host / 其他 agent 框架

一句话：

> app-server 是“直接操纵 Codex”；mcp-server 是“把 Codex 作为一个工具暴露出去”。

## 关键代码路径

- `codex-rs/app-server/src/message_processor.rs`
- `codex-rs/app-server/src/codex_message_processor.rs`
- `codex-rs/mcp-server/src/message_processor.rs`
- `codex-rs/mcp-server/src/codex_tool_runner.rs`
- `codex-rs/core/src/thread_manager.rs`

## 证据 1：两个 server 都自己 new `ThreadManager`

- app-server：`MessageProcessor::new()` 里 `Arc::new(ThreadManager::new(... SessionSource::...))`
- mcp-server：`MessageProcessor::new()` 里也 `Arc::new(ThreadManager::new(... SessionSource::Mcp ...))`

这说明它们都不是 runtime 本体，而是不同入口面。

## 证据 2：app-server 暴露的是“线程 API”

app-server 那边的典型动作是：

- `thread_manager.start_thread_with_tools_and_service_name(...)`
- `thread_manager.resume_thread_with_history(...)`
- `load_thread()` → `thread_manager.get_thread(...)`

然后围绕 thread 做：

- listener attach
- watch state
- outgoing event
- thread snapshot / config snapshot

这说明 app-server 的资源模型是：

> thread / conversation / connection / request

也就是 **原生 Codex 资源模型**。

## 证据 3：mcp-server 暴露的是“工具 API”

`mcp-server/src/message_processor.rs` 的 `tools/list` 只暴露两个主要工具：

- `codex`
- `codex-reply`

`tools/call` 里：

- `codex` → 启一个 Codex session
- `codex-reply` → 往已有 session/thread 继续喂 prompt

这说明 mcp-server 对外资源模型不是 thread API，而是：

> 一个可以被 LLM host 调用的工具

thread 只是 structured content 里回传给 host 的 continuation handle。

## 证据 4：mcp-server 的 runner 是“跑完整工具会话直到可返回”

`codex_tool_runner.rs` 做的事情很典型：

1. `thread_manager.start_thread(config)`
2. 把初始 prompt 包成 `Submission`
3. `thread.submit_with_id(...)`
4. 持续 `thread.next_event().await`
5. 处理 approval request / patch approval / error / turn complete
6. 最后按 MCP `CallToolResult` 形状回给 host

这套模型非常 MCP：

- host 发起 `tools/call`
- server 内部跑复杂工作流
- 中间事件可通知
- 最终仍要返回 `CallToolResult`

它不是通用控制平面，而是 **长生命周期工具执行器**。

## 证据 5：app-server-client 明确想统一“直接 client”语义，而不是 MCP host 语义

`app-server-client` 强调的是：

- in-process / remote 一致
- 保持 request / notification / event 模型
- 不暴露 core handle

这说明 app-server 的目标是打造 **Codex 自己的 embedding contract**。

而 mcp-server 的目标不是这个；它必须服从 MCP 协议的工具模型。

## 设计判断

### 1. 两者不是重复，而是两种产品包装

同样一套 core runtime，可以被包装成：

- 原生控制平面（app-server）
- 外部 agent 工具（mcp-server）

这很正常，而且是好事。

### 2. app-server 更“第一方”

因为它保留了：
- thread model
- listener/watch/state model
- richer control semantics

### 3. mcp-server 更“适配器化”

它为了适配 MCP host，需要把连续线程交互压成：
- `codex`
- `codex-reply`
- `CallToolResult`

这是典型的协议降维封装。

### 4. 所以后续别把 mcp-server 当成 app-server 的别名

这个判断不推荐。

更准确的理解应该是：

- **app-server = native control-plane surface**
- **mcp-server = toolified surface for external MCP ecosystems**

## 当前边界判断

- `app-server`：原生控制平面 / embedding server
- `app-server-client`：原生调用方 facade
- `mcp-server`：MCP 工具暴露面
- `codex_tool_runner`：MCP 工具会话执行器
- `ThreadManager`：两者共享的 runtime 入口

## 后续还值得继续挖的点

1. app-server 与 mcp-server 事件模型的差异到底有多大
2. app-server 是否会成为未来更统一的外部嵌入面，而 mcp-server 只保留生态适配角色
3. `codex-reply` 这种 continuation 设计对外部 host 来说是否足够稳定
