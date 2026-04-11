# Codex 源码拆解 20：TUI 向 app-server 收敛的现状

## 这篇看什么

回答：**TUI 现在到底已经多大程度收敛到 app-server contract？还有哪些地方仍然直接咬 core？**

## 先给结论

当前 TUI 的状态可以概括成一句话：

> 主执行面已经高度 app-server 化，但仍保留一圈 legacy core helper 作为迁移边缘层。

也就是说，不是“还没迁”，而是：
- 主干大多已经切过去了
- 边角和本地工具类能力还没完全抽成 RPC

## 关键代码路径

- `codex-rs/tui/src/app_server_session.rs`
- `codex-rs/tui/src/lib.rs`
- `codex-rs/tui/src/app/app_server_adapter.rs`
- `codex-rs/tui/src/app/app_server_requests.rs`
- `codex-rs/tui/src/chatwidget.rs`

## 已经明显收敛到 app-server 的部分

### 1. session / thread 生命周期
`AppServerSession` 已经统一封装：

- `start_thread`
- `resume_thread`
- `fork_thread`
- `thread_list`
- `thread_read`
- `turn_start`
- `turn_interrupt`
- `turn_steer`

这说明 TUI 主会话面已经不是直连 core，而是：

- embedded app-server
- remote app-server

两态统一。

### 2. embedded / remote 都是一等路径
`AppServerTarget::{Embedded, Remote}` 说明：

- 不是“本地 TUI = 真模式，remote = 补丁模式”
- 而是从抽象上已经接受 app-server 是共同控制面

### 3. live event handling
`app_server_adapter.rs` 和 `chatwidget.rs` 已经直接消费：

- server notifications
- server requests

包括：
- turn / approval / plugin / MCP / account 等事件

也就是说，前台 UI 的实时驱动主链，已经很大程度建立在 app-server protocol 上。

### 4. plugin / MCP / account 等控制面能力
TUI 侧已通过 RPC 拿：

- plugin list/read/install/uninstall
- MCP server statuses
- account rate limits

这些都说明 app-server 不只是拿来跑聊天，而是在扩成 TUI 的真正 control-plane backend。

## 还没完全收敛的部分

### 1. legacy_core 仍然大量存在
`app-server-client` 里甚至还显式保留了：

- `legacy_core`

文档也写得很清楚：

- 新行为应优先走 app-server protocol
- 这个模块是为了让旧 startup/config path 逐步迁移

这几乎等于官方自己承认：

> 迁移进行中，但还没完全收口。

### 2. config / 本地状态 / 文件系统 helper 仍有直连
TUI 里仍直接用 core 的东西处理：

- config load / edit
- 本地 message history
- 一些 rollout/session meta 读取
- onboarding / trust directory / OSS provider 设置
- Windows sandbox 相关 helper

这些大多不是“agent 主执行路径”，而是：
- 本地环境配置
- 辅助功能
- 启动期能力

### 3. resume picker 还是 hybrid
resume picker 既有：
- app-server 路径
- 也仍保留 rollout-based legacy path

这说明历史数据浏览这块，还没完全统一成 app-server-only。

### 4. 不是所有 server request 类型都完全支持
在 app-server request tracking 那边，TUI 还明确 reject 了一些：

- `DynamicToolCall`
- 某些 legacy approval request 形态

这说明协议面已经很大，但客户端适配还不是 100% 全量覆盖。

## 当前最准确的判断

### 不是“是否收敛”问题，而是“收敛到了什么程度”
当前答案是：

- **核心 thread/session/live-event 主链：高度收敛**
- **本地 helper/config/历史浏览边缘层：仍 hybrid**

### 这也解释了为什么 TUI 看起来有点“双栈”
因为它现在不是：
- 纯旧架构
- 或纯新架构

而是：
- 新主链已经站住
- 旧边缘还没完全迁完

## 一个更准确的分层图

### 新主链
TUI UI
→ `AppServerSession`
→ embedded / remote app-server
→ `ThreadManager` / core runtime

### 旧边缘
TUI local helpers
→ `legacy_core` / config / rollout / local filesystem helpers

## 设计判断

### 1. app-server 已经是 TUI 的主控制面方向
这个判断现在足够稳。

### 2. legacy_core 的存在不是失败，而是迁移缓冲层
这个非常正常，反而比硬切一刀安全。

### 3. 真正还没完全迁完的，多半不是 agent runtime 主流程，而是周边本地能力
这也是合理顺序。

先把：
- thread lifecycle
- event streaming
- approvals
- remote support

迁过去，再慢慢吃本地工具层。

## 当前边界判断

- `AppServerSession`：TUI 的主控制面客户端
- `app_server_adapter`：协议适配/过渡层
- `chatwidget`：app-server notification/request 消费面
- `legacy_core`：尚未迁移完的边缘 helper 汇总

## 还值得继续观察的点

1. 哪些 legacy core helper 最可能被下一批 RPC 化
2. resume picker 何时彻底切成 app-server 路径
3. TUI 是否最终会把本地-only helper 也全包到 app-server 里，还是长期保留 hybrid 结构
