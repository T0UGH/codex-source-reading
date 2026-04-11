# Codex 源码拆解 10：app-server 到 ThreadManager 的桥

## 这篇看什么

回答：**app-server 收到 JSON-RPC 之后，究竟怎么落到 core 的线程 runtime？**

## 先给结论

`app-server` 自己不是 runtime。

它更像一层 **协议入口 + 会话编排层**，真正的线程生命周期仍然落在 `core::ThreadManager`：

1. `app-server::MessageProcessor::new()` 创建共享 `ThreadManager`
2. 再把它塞进 `CodexMessageProcessor`
3. 具体的 `thread/start` / `thread/resume` / `load_thread` / `mcp tool call` 都通过 `thread_manager` 落到 core
4. app-server 额外补一层：listener / watch / outgoing / connection 维度的状态管理

一句话：

> app-server 是控制面，ThreadManager 是线程 runtime 入口面。

## 关键代码路径

- `codex-rs/app-server/src/message_processor.rs`
- `codex-rs/app-server/src/codex_message_processor.rs`
- `codex-rs/core/src/thread_manager.rs`
- `codex-rs/app-server-client/src/lib.rs`

## 证据 1：app-server 启动时先造 ThreadManager

`app-server/src/message_processor.rs` 里：

- 创建 `AnalyticsEventsClient`
- `Arc::new(ThreadManager::new(...))`
- 再把这个 `thread_manager` 注入 `CodexMessageProcessorArgs`

这说明 app-server 没有自己再造一套线程 runtime。

它从一开始就承认：
- 线程创建
- 线程查找
- 共享 manager（models / skills / plugins / mcp / env）

都应该归 `ThreadManager`。

## 证据 2：CodexMessageProcessor 只是“协议方法 → runtime 方法”映射层

`CodexMessageProcessor` 内部核心字段：

- `thread_manager`
- `thread_state_manager`
- `thread_watch_manager`
- `command_exec_manager`
- `outgoing`

这里能看出职责拆分：

- `thread_manager`：真正线程对象的 owner / creator
- `thread_state_manager`：偏 app-server 本地状态辅助
- `thread_watch_manager`：给连接/UI 做线程状态观测与缓存
- `outgoing`：往 JSON-RPC / event 流回写

也就是说，**app-server 在 ThreadManager 外面又包了一层“对客户端友好的状态壳”**。

## 证据 3：thread/start 直接打到 ThreadManager

`codex_message_processor.rs` 的启动路径里，核心调用是：

- `thread_manager.start_thread_with_tools_and_service_name(...)`

返回 `NewThread { thread_id, thread, session_configured }` 之后，app-server 再做这些事：

- 设置 app-server client info
- 读取 `config_snapshot`
- 构造 protocol 层 thread snapshot
- `ensure_conversation_listener_task(...)`
- `thread_watch_manager.upsert_thread...`
- resolve 当前 thread status

这个顺序非常说明问题：

**线程先在 core 里出生，app-server 再把它包装成可订阅、可展示、可回放的 server-side object。**

## 证据 4：resume 也不是 app-server 自己恢复，而是委托 core

恢复路径里核心调用是：

- `thread_manager.resume_thread_with_history(...)`

然后 app-server 再：

- 自动 attach listener
- `load_thread_from_resume_source_or_send_internal(...)`
- `thread_watch_manager.upsert_thread(...)`
- 修正状态 / stale turn

这说明：

- **历史恢复语义** 在 core
- **连接态恢复 / 观察态恢复** 在 app-server

这是很健康的边界。

## 证据 5：load_thread 只是从 manager 拿 handle

`load_thread(thread_id)` 本质上就是：

1. 把字符串 thread id 转成 core 的 `ThreadId`
2. `thread_manager.get_thread(thread_id).await`

这进一步说明 app-server 并不拥有 thread object 的真相源。

## app-server-client 的意义

`app-server-client/src/lib.rs` 里写得很直白：

- in-process client 也尽量保留 app-server request/notification/event 模型
- 不直接暴露 core runtime handle
- 这样 in-process / remote client 行为能尽量一致

这是一个很成熟的判断：

> 就算同进程，也别直接把 UI 和 core 搞成强耦合。

所以当前 TUI / exec 更像是：
- **逻辑上走 app-server contract**
- **物理上可走 in-process**

## 设计判断

### 1. app-server 不是 runtime owner，而是 runtime facade

不推荐把 app-server 理解成“另一个 core”。

更准确的说法是：

- `ThreadManager` 持有线程与共享 manager
- `CodexThread` 暴露线程 API
- `app-server` 把这些对象投影成 JSON-RPC surface

### 2. app-server 在补“连接态能力”

core 的世界是线程与事件。
app-server 的世界还要多：

- connection id
- request id
- listener attach/detach
- watch cache
- outgoing transport

所以 app-server 不是薄壳，而是 **控制面适配层**。

### 3. in-process client 仍坚持协议模型，是长期可演进设计

这点很关键。

很多系统会在 in-process 模式直接偷懒暴露内部对象，最后 remote / embedded 两套模型分叉。

Codex 这里明显在避免这个坑。

## 当前边界判断

- `ThreadManager`：线程 runtime 的创建/持有/共享 manager 入口
- `CodexThread`：线程级 handle
- `app-server::MessageProcessor`：server 级总装配
- `CodexMessageProcessor`：协议方法分发与 server-side 编排层
- `thread_watch_manager` / `thread_state_manager`：app-server 自己补的连接态状态层
- `app-server-client`：调用方 facade，不让上层直接咬 core

## 一个更准确的系统图

`JSON-RPC request`
→ `app-server::MessageProcessor`
→ `CodexMessageProcessor`
→ `ThreadManager`
→ `CodexThread`
→ `Codex(Session/Turn loop)`

然后事件再反向回：

`CodexThread event`
→ listener/watch/outgoing
→ app-server protocol event
→ client/TUI/SDK

## 后续还值得继续挖的点

1. `ensure_conversation_listener_task` 到底如何把 thread event 投影成 app-server event
2. `thread_watch_manager` 和 `thread_state_manager` 的边界还能再拆细
3. TUI 现在到底有多少逻辑已经切到 app-server contract，有多少还在走 core 直连
