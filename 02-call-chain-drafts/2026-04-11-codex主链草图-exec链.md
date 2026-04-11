# Codex 主链草图：exec 非交互链

## 当前版本性质

这是文本版草图，不是最终定稿。

## 当前主链判断

```text
codex exec
  → codex-rs/cli/src/main.rs
  → codex_exec::run_main
  → build config / logging / app-server client args
  → InProcessAppServerClient::start
  → ThreadStart / ThreadResume
  → TurnStart / ReviewStart
  → consume app-server notifications
  → EventProcessor
  → human stderr output or JSONL output
```

## 当前最关键的理解

1. `exec` 不是直接绕开系统调用 core，而是 headless client over app-server
2. exec 与 TUI 共享很多 runtime 核心，只是前端形态不同
3. output processor 是 exec 很关键的一层，它决定人类输出和 JSONL 输出两种消费面

## 还没补齐的点

- review 路径是否和 exec 主线完全共用，只是 turn type 不同
- exec 内部和 `core::exec` / `unified_exec` 的更深关系还可继续拆
