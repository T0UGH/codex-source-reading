# Codex 主链草图：interactive TUI 链

## 当前版本性质

这是文本版草图，不是最终定稿。

## 当前主链判断

```text
codex
  → codex-cli/bin/codex.js
  → native Rust binary
  → codex-rs/cli/src/main.rs
  → default interactive path
  → codex_tui::run_main
  → choose embedded/remote app-server
  → App::run
  → AppServerSession::start_thread / turn_start / turn_steer
  → app-server side
  → core::ThreadManager
  → CodexThread
  → Codex
  → sampling / tool runtime / event stream
  → app-server notifications
  → TUI render loop
```

## 当前最关键的理解

1. TUI 不是直接对 core 发裸调用，而是先走 app-server 协议面
2. `App` 是 UI orchestration，不是 agent loop 本身
3. `core` 仍是实际 runtime 主体

## 还没补齐的点

- app-server 内部如何把 `ThreadStart` / `TurnStart` 进一步桥接到 `ThreadManager`
- TUI 内两套 event source 的关系
