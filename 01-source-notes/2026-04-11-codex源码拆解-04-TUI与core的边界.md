# Codex 源码拆解 04：TUI 与 core 的边界

## 这篇看什么

回答一个关键问题：**`codex-rs/tui` 到底是系统本体，还是只是交互前端？它和 `core` 之间隔着什么？**

## 先给结论

`codex-rs/tui` 不是 Codex 的 runtime 本体，而是一个 **interactive front-end + bootstrap layer**。

它负责：
- 终端 UI 与事件循环
- 启动 embedded 或 remote app-server
- 把用户动作翻译成 app-server 协议请求
- 消费线程/turn 事件并渲染

真正的 agent runtime 主体仍在 `core`。

更准确地说：

```text
TUI
  → app-server client/session
  → app-server protocol / service
  → core::ThreadManager / CodexThread / Codex
```

## 关键代码路径

- `codex-rs/tui/src/main.rs`
- `codex-rs/tui/src/lib.rs`
- `codex-rs/tui/src/app.rs`
- `codex-rs/tui/src/app_server_session.rs`
- `codex-rs/tui/src/tui.rs`
- `codex-rs/core/src/lib.rs`
- `codex-rs/core/src/thread_manager.rs`
- `codex-rs/core/src/codex_thread.rs`
- `codex-rs/core/src/codex.rs`

## 关键证据

### 1. TUI 启动并不直接落到 core API
- `tui/src/main.rs` 只是解析参数后调用 `codex_tui::run_main`
- `tui/src/lib.rs` 会先决定 app-server target：`Embedded` 或 `Remote`
- `start_app_server(...)` 明确存在 embedded/remote 两条 app-server 路径

这说明 TUI 先处理的是 **服务目标选择**，不是直接 new 一个 core runtime。

### 2. TUI 通过 `AppServerSession` 发请求
`app_server_session.rs` 里可以看到：
- `ThreadStart`
- `TurnStart`
- `TurnSteer`

也就是说，TUI 不是直接对 `ThreadManager` 调方法，而是先走 **app-server protocol**。

### 3. TUI 自身的职责偏 UI orchestration
`app.rs` 是一个非常大的 UI orchestration 模块：
- 启动/恢复线程
- 管理 widget
- 处理 prompt submit
- 处理 interrupt / steer / redraw / status

`tui.rs` 则更偏：
- terminal 管理
- draw 调度
- event stream
- alt-screen / repaint

### 4. core 才是 runtime 聚合核心
`core/src/lib.rs` 导出了：
- `ThreadManager`
- `CodexThread`
- `McpManager`
- skills / plugins / sandboxing / rollout / state bridge

而 `thread_manager.rs` 与 `codex_thread.rs` 进一步说明：
- `ThreadManager` 管全局线程/服务资源
- `CodexThread` 管单线程交互面
- `Codex` 管更底层的 session loop / submission / event transport

## 初步主链判断

### interactive TUI 主链（当前证据版）
```text
codex
  → cli/main.rs
  → codex_tui::run_main
  → choose embedded/remote app-server
  → App::run
  → AppServerSession::start_thread / turn_start / turn_steer
  → app-server side
  → core::ThreadManager / CodexThread / Codex
```

## 设计判断

### 1. TUI 是“前端”，不是“核心”
这一点和很多人直觉相反。

如果只看 `codex` 命令，会误以为 TUI 就是整个系统；
但从源码关系看，TUI 更像一个正式交互客户端。

### 2. app-server 协议是 TUI 和 core 之间的重要中介层
这很关键，因为它意味着：
- UI 可以替换
- embedding 能力可以共享
- runtime 核心不需要直接耦合在 terminal UI 上

### 3. `app.rs` 很重要，但不能误读成 runtime 主线
它更多是 **UI orchestration**，不是 agent reasoning loop 本身。

## 当前边界判断

- `tui`：交互前端 + UI 编排层
- `app_server_session`：TUI 到 app-server 的 typed bridge
- `app-server`：控制平面 / 协议服务层
- `core`：真实 runtime 主体

## 还没确认的点

1. app-server 到 `ThreadManager` 的中间桥接细节还没单独拆。
2. TUI 里同时存在 app-server 事件和 active thread event channel，这两套信号流的关系还要继续确认。
3. embedded 与 remote app-server 在能力面上是否完全等价，还没验证。

## 下一步建议

继续写：
- `core 作为 runtime 聚合核心`
- `interactive TUI 主链草图`
- `app-server 在体系里的角色`
