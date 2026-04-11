# Codex 源码拆解 03：Codex CLI 总入口与命令分流

## 这篇看什么

回答：**`codex` 命令本身是怎么组织的？默认为什么会进入 TUI？还有哪些重要模式？**

## 先给结论

`codex-rs/cli/src/main.rs` 是 Codex 的统一命令入口。

它不是只有一个聊天模式，而是一个 **multitool CLI**：
- 默认无子命令时走 interactive TUI
- 有子命令时分流到 exec / review / mcp / app-server / sandbox / cloud / resume / fork 等模式

所以 `codex` 不是单一命令，而是一个 runtime front door。

## 关键代码路径

- `codex-rs/cli/src/main.rs`
- `codex-rs/cli/src/lib.rs`
- `codex-rs/tui/src/lib.rs`
- `docs/exec.md`

## 关键证据

### 1. 顶层 CLI 结构
`main.rs` 里的 `MultitoolCli`：
- 带 `interactive: TuiCli`
- 带 `subcommand: Option<Subcommand>`

这说明：
- interactive 不是某个单独子命令才有
- 它就是默认正路径的一部分

### 2. 主要子命令
当前能看到的重要 subcommands 包括：
- `Exec`
- `Review`
- `Login`
- `Logout`
- `Mcp`
- `Marketplace`
- `McpServer`
- `AppServer`
- `App`（macOS）
- `Sandbox`
- `Debug`
- `Apply`
- `Resume`
- `Fork`
- `Cloud`
- `ResponsesApiProxy`
- `ExecServer`
- `Features`

### 3. TUI 不是孤立工具
`codex-rs/tui/src/lib.rs` 显示 TUI 层大量依赖：
- `codex_app_server_client`
- `codex_app_server_protocol`
- `codex_exec_server`
- `codex_login`
- `codex_state`
- `legacy_core`（即 core 能力）

这说明 TUI 是上层交互界面，不是整个系统本身。

## 初步命令分流判断

### 默认路径
```text
codex
  → cli main
  → interactive TUI path
```

### 非交互路径
```text
codex exec
  → exec path
```

### 其他 runtime surfaces
- `codex review`
- `codex mcp`
- `codex mcp-server`
- `codex app-server`
- `codex sandbox`
- `codex resume`
- `codex fork`
- `codex cloud`

## 设计判断

### 1. Codex CLI 是多运行面入口，不只是聊天入口
这点很关键。

它已经同时承载：
- interactive use
- non-interactive execution
- review
- external MCP management
- app-server / embedding related surfaces
- cloud task related surfaces

### 2. 默认走 TUI 说明 OpenAI 认为交互式 agent 是主体验
而 `exec` 更像一个重要补充，而不是唯一主线。

### 3. `cli` crate 更像入口编排层
它负责：
- 命令解析
- 模式分流
- 把用户带到不同 runtime surface

而不是负责全部业务逻辑。

## 当前边界判断

- `codex-rs/cli`：统一命令入口 / 模式分流层
- `codex-rs/tui`：interactive front-end
- `codex-exec`：非交互执行面
- `core`：更底层的 runtime 能力集合

## 还没确认的点

1. interactive TUI 进入 `core` 的真实调用链细节还没展开
2. TUI 与 app-server 的关系仍需继续拆：是强依赖还是部分模式依赖
3. `review` 是否共享大部分 exec/runtime 基础设施，还没继续追

## 下一步建议

继续拆：
1. `codex-rs/tui/src/lib.rs`
2. `codex-rs/core/src/lib.rs`
3. `codex-rs/exec/src/lib.rs`
4. `docs/exec.md`
