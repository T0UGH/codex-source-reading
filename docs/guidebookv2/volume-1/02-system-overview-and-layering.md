---
title: Codex guidebook 01｜系统总图与分层
date: 2026-04-12
status: draft
---

# 01｜系统总图与分层

## 先给结论

如果只用一句话概括 Codex 的系统形态，那就是：

> **一个由 CLI/TUI/server 多个入口包着的 runtime 系统。真正的主体不在分发壳里，而在 `codex-rs` 里的 `core`、`app-server`、`mcp-server` 以及它们围绕 `ThreadManager` 形成的运行面。**

这意味着，理解 Codex 不能从 npm 包、外层命令壳或目录树开始，而要先回答两个问题：

1. 哪一层只是分发和命令分流？
2. 哪一层真的持有 thread、session、turn 和执行中的 runtime 状态？

这章就回答这两个问题。

---

## 1. 先把壳和主体分开

Codex 在用户侧最先暴露出来的，是一个统一命令入口。但从源码上看，这个入口更像一个**命令壳**，不是主要 runtime。

`cli/src/main.rs` 负责两件事：

- 定义命令面
- 把请求分发到不同产品面或服务面

它能把调用导向：

- 交互式 TUI
- 非交互式 exec / review
- app-server
- mcp-server
- exec-server

所以这一层的职责，不是持有复杂运行态，而是：

> **把不同使用方式接到后面的真正系统上。**

这也是为什么 `codex-cli/` 更适合被看成分发壳和入口壳，而不是“系统主体”。

---

## 2. `codex-rs/cli` 是统一入口层，不是终极 runtime owner

壳下面的第一层，是 `codex-rs/cli`。

它和外层包装的差别在于：

- 外层主要负责分发和用户入口体验
- `codex-rs/cli` 已经进入真正的命令语义和模式分流

但它依然不是最深的 runtime owner。更准确地说，它是：

> **统一入口层和模式分发层。**

也就是说，它会决定当前请求应该走：

- TUI 交互模式
- app-server 模式
- mcp-server 模式
- exec-server 模式

但 thread 真正如何创建、session 如何运行、事件如何推进，并不在这里完成。

---

## 3. TUI 是交互面，不是 runtime 核心

如果只从产品感知上看，很多人会把 Codex 理解成一个 TUI 工具。但源码显示，TUI 更准确的定位是：

> **面向人的交互前端。**

它负责：

- 渲染界面
- 输入处理
- 本地 UI 状态
- 启动 / 恢复 / fork 等用户动作

但 TUI 并不直接把自己做成一套完整 runtime。现在的主路径已经更接近：

> **TUI over app-server**

也就是：

- TUI 通过 `AppServerSession` 和 app-server 对话
- 这个 session 既可以连嵌入式 app-server
- 也可以连 remote app-server

因此，TUI 更应该被理解成一个 product frontend，而不是 runtime owner。

---

## 4. `core` 才是 runtime 聚合中心

从职责上看，`core` 是 Codex 最接近“系统主体”的地方。

这里聚合了：

- thread 生命周期
- session 启动
- model / tool / auth / exec 等运行期服务
- rollout recorder 和状态恢复相关接入
- unified-exec 等关键子系统

尤其重要的是：`ThreadManager` 在这里。

`ThreadManager` 的角色，不是简单的 registry，而更接近：

> **live thread runtime 的创建与持有入口。**

这点很关键，因为它直接决定了一个核心判断：

- app-server 不是 runtime 真正 owner
- `ThreadManager` 更接近 runtime 真正 owner

如果要找“哪个地方真正拥有 live thread memory 和 thread lifecycle”，答案更接近 `core`，而不是上面的各种入口层。

---

## 5. app-server 是控制面 facade，不是另一套 runtime

app-server 很容易被误读成“Codex 的主服务端”，但源码关系更像：

> **一个建立在 core runtime 之上的控制面 facade。**

它的职责是把下面的 thread/session/runtime 语义，投影成更稳定的对外协议面。它负责的重点包括：

- 请求路由
- connection 管理
- thread/listener 绑定
- thread/turn 状态对外投影
- start / resume / fork / replay 这类控制面动作

所以它很强、很厚，但它并不是重新实现一套 runtime，而是在 core 上面形成一个面向客户端、SDK、TUI 和远端调用者的控制面。

更直白地说：

- `core` 负责让 thread 真正活起来
- `app-server` 负责让外界稳定地使用、观察和控制这些 thread

这也是为什么 TUI 会越来越向 app-server 收敛：不是因为 app-server 比 core 更“底层”，而是因为它更适合作为统一交互入口。

---

## 6. mcp-server 不是 app-server 的另一种皮，而是另一条暴露线

`mcp-server` 和 `app-server` 都建立在 core 之上，但它们不是同一层产品，只是换了协议。

更准确的分法是：

- **app-server**：通用控制面
- **mcp-server**：把 Codex 作为 MCP 能力暴露出去的协议适配层

也就是说，mcp-server 的目标不是承接完整 thread/session 控制面，而是：

> **把 Codex 包成一个面向外部系统的 MCP 服务。**

所以它和 app-server 的关系，不是“一个本地，一个远端”，而是“两个不同的 exposure layer”。

---

## 7. remote app-server 不是另一套系统，只是 transport variation

Codex 里还有一个很容易被误会的点：remote app-server。

如果只看名字，会觉得它像另一套服务端实现。但从 TUI 侧抽象看，它更像：

> **同一套 app-server contract 的远端传输形态。**

对 TUI 来说，embedded app-server 和 remote app-server 最终都被包进 `AppServerSession` 这一层里。也就是说：

- 对上层 UI 来说，它们尽量表现为同一个会话接口
- 对系统分层来说，它们的主要差异在 transport placement，而不是 runtime semantics

因此，remote app-server 最合理的定位不是“平行系统”，而是：

> **同一控制面 contract 的远端部署/连接变体。**

---

## 8. 这一整套系统到底怎么分层

到这里，可以把 Codex 粗略拆成五层。

### 第一层：分发壳 / 命令壳
- `codex-cli/`
- npm / 包装层
- Rust CLI 外层入口

职责：给用户一个命令入口，承接安装和分发。

### 第二层：统一入口与模式分流层
- `codex-rs/cli`

职责：决定请求进入哪种产品面或服务面。

### 第三层：前端 / 暴露面
- TUI
- app-server
- mcp-server
- remote app-server
- exec-server

职责：把系统以不同交互方式暴露出来。

### 第四层：runtime 聚合层
- `core`
- `ThreadManager`
- Codex session/runtime 相关核心对象

职责：真正持有 live thread、session 和主要运行逻辑。

### 第五层：持久化与底层子系统
- rollout
- state DB / SQLite
- exec / unified-exec
- model / tool / auth 等更深层实现

职责：提供 durability、执行、恢复和能力接入。

---

## 9. 这一章最重要的边界判断

读完这一章，应该先稳定这几个判断。

### 判断 1：`codex-cli/` 不是主体
它是用户入口和分发壳，不是系统 runtime 主体。

### 判断 2：`codex-rs/cli` 是统一入口层
它负责模式分流，但不是最终 runtime owner。

### 判断 3：TUI 是前端，不是 runtime 核心
它越来越通过 app-server 使用底层系统，而不是自己承载完整 runtime。

### 判断 4：`core` 是 runtime 聚合中心
真正的 live thread / session 语义更靠近这里。

### 判断 5：`ThreadManager` 比 app-server 更接近 runtime owner
app-server 负责控制面，不负责重新定义一套 runtime。

### 判断 6：app-server 是 facade，不是平行 runtime
它把 core runtime 投影成稳定协议面。

### 判断 7：mcp-server 是另一条 exposure layer
它不是 app-server 的简化版，而是 MCP 方向的对外暴露层。

### 判断 8：remote app-server 是同一 contract 的 transport variation
不是另一套独立系统。

---

## 10. 下一章该接什么

系统总图定完之后，最自然的下一步不是立刻钻 listener 或 unified-exec，而是先回答一个更基础的问题：

> **Codex 为什么能把 thread、history 和会话状态找回来？**

所以，下一章应该接：

- rollout 是什么
- SQLite state DB 是什么
- config、checkpoint、backfill 各自扮演什么角色
- 恢复为什么本质上更像 replay，而不是“查库拼对象”

也就是下一章的主题：**状态、持久化与恢复。**

---

## 相关阅读

- 想继续看状态如何持久化与恢复：[[02-状态持久化与恢复]]
- 想直接进入 app-server 控制面主线：[[03-app-server与thread-turn主线]]
- 想先建立全仓关键入口与函数地图：[[11-关键函数索引]]、[[12-调用链索引]]

## 关键文件

- `codex-rs/cli/src/main.rs`
- `codex-rs/tui/src/lib.rs`
- `codex-rs/tui/src/app_server_session.rs`
- `codex-rs/tui/src/app.rs`
- `codex-rs/app-server/src/message_processor.rs`
- `codex-rs/mcp-server/src/lib.rs`
- `codex-rs/mcp-server/src/message_processor.rs`
- `codex-rs/core/src/lib.rs`
- `codex-rs/core/src/thread_manager.rs`
