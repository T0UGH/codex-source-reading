---
title: Codex guidebook 03｜app-server 与 thread-turn 主线
date: 2026-04-12
status: draft
---

# 03｜app-server 与 thread-turn 主线

## 先给结论

如果说 `core` 负责让 thread 真正活起来，那么 `app-server` 负责的事情就是：

> **把 core runtime 的 live event、thread 状态和 turn 语义，持续投影成一个可连接、可恢复、可 replay、可对外消费的控制面协议。**

这意味着，app-server 的关键价值不在“再做一套 runtime”，而在于：

- 如何把 thread 绑定到 listener
- 如何把 runtime event 变成稳定协议通知
- 如何维护 thread 局部状态和对外状态
- 如何在 resume / reconnect 时保证顺序、状态和请求重放都不乱

这章就讲这条主线。

---

## 1. app-server 拥有什么，不拥有什么

先做最重要的边界判断。

app-server 确实很厚，但它并不是 thread 真正 owner。真正的 thread 运行时仍然在 core 的 `ThreadManager` 那边。

app-server 自己持有的，主要是这些东西：

- `thread_manager`：指向 core runtime
- `thread_state_manager`：listener 局部运行态
- `thread_watch_manager`：对外 thread status 投影态
- `outgoing`：请求 / 响应 / 通知的发射面
- connection / subscription 等控制面对象

所以最准确的说法是：

- **thread 执行真相**在 core
- **thread 协议投影真相**在 app-server

这就是 app-server 的角色：**thread 语义投影层 + control plane facade**。

---

## 2. 一条 thread 为什么要配一条 listener

理解 app-server 的主线，最关键的对象不是某个 API，而是 listener task。

每个活跃 thread，在 app-server 侧都可以挂一个 listener。这个 listener 不是简单“转发消息”的线程，而是：

> **thread 事件泵 + 协议投影入口 + 顺序控制点。**

它会持续监听：

- cancel 信号
- core conversation 的下一条 event
- listener command channel

也就是说，它一边接收 core runtime 的真实事件流，一边还负责处理额外命令，比如：

- running thread 的 resume 响应
- pending request 的 resolve
- 其他必须和 event 流串行处理的控制动作

这也是为什么 listener 这么关键：

> **很多正确性，不是靠某个共享锁保证，而是靠“所有关键动作都回到同一个 listener context 串行处理”保证。**

---

## 3. 为什么要有 listener generation

thread listener 不是永久不变的。连接可能重建，listener 也可能被替换。

`ThreadState` 里保存了 listener 相关的核心局部状态，比如：

- `cancel_tx`
- `listener_generation`
- `listener_command_tx`
- `current_turn_history`
- `listener_thread`

其中最重要的保护机制之一，就是 `listener_generation`。

它的意义是：

- 新 listener 装上时，generation 增长
- 旧 listener 退出时，只能在 generation 还匹配时清理自己
- 如果期间已经换过 listener，旧任务就不能把新 listener 的状态误删

这其实是在解决一个很现实的问题：

> **同一个 thread 的 listener 可能被重装，旧任务退出时不能误伤新任务。**

所以 generation 机制本质上是 app-server 这套控制面里的防 stale-task 设计。

---

## 4. listener 主循环到底在做什么

listener task 的主循环可以概括成一句话：

> **拿到 core event，先更新 thread 局部语义状态，再决定向哪些连接投影成什么协议事件。**

这个顺序非常重要。

它不是先把 event 发出去，再说别的；而是：

1. 锁住 thread state
2. 把 event 先喂给当前 turn/history 的局部语义状态
3. 读取 raw event opt-in 等 thread-local 配置
4. 拿到当前订阅这个 thread 的连接列表
5. 构造 thread-scoped outgoing sender
6. 调 `apply_bespoke_event_handling(...)` 做协议投影

这里的关键点在于：

- thread-local state 先更新
- 协议翻译后发生

这是为了避免一种很典型的错乱：协议通知已经发出去了，但 thread 局部状态还没同步，导致 resume、raw events、active turn snapshot 等信息对不上。

---

## 5. `apply_bespoke_event_handling` 为什么这么重要

这条主线上最关键的函数之一，是 `apply_bespoke_event_handling(...)`。

它的角色不是“if/else 很多的事件处理函数”这么简单，而更像：

> **从 core `EventMsg` 到 app-server 协议语义的总投影器。**

比如当 listener 收到 `TurnStarted` 时，它不会只是原样广播一个“开始了”的消息，而是会同时处理几件事：

- 中止或清理某些 pending server requests
- 更新 watch state，让 thread 进入 active
- 基于 `active_turn_snapshot()` 组出合适的通知体
- 向已订阅连接发出 `TurnStartedNotification`

`TurnComplete`、`TurnAborted` 也是类似：

- 既要处理 thread 状态变化
- 也要处理协议层该如何展示这次 turn 已经结束、失败或中断

所以它不是简单转发，而是在做：

> **runtime signal → client-facing semantic notification**

这也是 app-server 作为 facade 的真正重量所在。

---

## 6. 为什么 `thread_state_manager` 和 `thread_watch_manager` 要拆开

这是这条链路里最值得讲清楚的一个设计点。

### `ThreadStateManager`
它更偏 listener 内部运行态。它管的是：

- 哪些连接订阅了哪个 thread
- 每个 thread 当前 listener 是谁
- command channel 在哪
- 当前 active turn 的 builder 状态
- raw events opt-in 等 thread-local 细节

它要回答的问题是：

> **这个 thread 的 listener 现在在做什么、还能接受什么命令、局部状态长什么样？**

### `ThreadWatchManager`
它更偏对外 thread status 投影。它管的是：

- running 与否
- permission request 是否挂起
- user input request 是否挂起
- system error 是否存在
- 当前 thread 对外应该显示成 Active / Idle / NotLoaded / SystemError

它要回答的问题是：

> **客户端现在应该怎么看待这个 thread？**

### 为什么必须拆
如果把两者混在一起，就会出现两个问题：

1. UI-facing 状态会和 listener 内部细节过度耦合
2. reconnect/resume 时，局部机制状态和展示状态会互相污染

所以这两个 manager 的分离，本质上是把：

- **内部协调状态**
- **对外展示状态**

分成了两层。

这也是 app-server 这套设计里非常成熟的一点。

---

## 7. `active_turn_snapshot` 为什么是关键桥点

thread 要恢复、resume 或对外展示，不可能每次都只看持久化历史。因为对于正在运行的 thread，总有一部分最新状态只存在于内存里。

这时 `active_turn_snapshot()` 就变成关键桥点。

它依赖 `ThreadHistoryBuilder` 维护的当前 turn 语义状态，可以返回：

- 正在进行中的 turn
- 或最近一个已完成 turn 的合适投影

listener 在每次 event 到来时，都会先更新这套 builder。所以它实际上承担了这样一个职责：

> **把 live event 流中的“当前 turn 语义”保留下来，供 resume、通知和视图装配使用。**

后面 running-thread resume 能把 persisted rollout 历史和 live active turn 拼起来，靠的就是这个桥点。

---

## 8. running-thread resume 为什么必须走 listener command channel

这是 app-server 主线上最关键的一个正确性设计。

如果一个 thread 还在运行，而客户端要 resume 它，系统不会简单地在外面拼个响应就发回去。它会把这个动作封装成：

- `SendThreadResumeResponse(...)`

然后塞回当前 listener 的 command channel，让 listener 在自己的上下文里处理。

这样做的原因很明确：

- resume 响应不仅要发历史
- 还要在正确时刻把连接重新订阅进后续事件流
- 还可能要 replay pending requests
- 这些动作必须和当前 event 流保持顺序一致

也就是说，resume 的关键不是“能不能拼出一份历史”，而是：

> **能不能在不打乱事件顺序的前提下，把历史、当前活跃态、待处理请求和后续订阅重新接起来。**

这就是 listener context 串行化存在的意义。

---

## 9. pending request replay 不是传输重试，而是 thread 语义的一部分

app-server 里还有一条很容易被低估的链：pending server request replay。

当某个 thread 上有尚未 resolved 的 request，比如 approval 或 user input 请求时，系统会把它们按 thread scope 存下来。

这样一来，如果连接断开、客户端重连或 resume：

- app-server 可以把这批 pending requests 找出来
- 按顺序 replay 给新连接

更关键的是，请求的 resolve 也不是随手发个 resolved 就结束，而是会再通过 listener command channel 进入 listener 上下文，保证 resolved 通知和 request replay 之间的顺序正确。

所以这一套机制不是“网络不稳时补发一下”，而是：

> **thread runtime 语义本来就允许 request 跨 reconnect/resume 继续存活。**

这也是 app-server 比简单事件转发器更像 control plane 的地方。

---

## 10. 为什么 app-server 里到处都是“小修正器”

app-server 的代码很容易给人一种印象：为什么这么多小 helper、小校正器、小 reconcile 函数？

答案其实很统一：

> **因为它要把 volatile runtime、持久化历史和客户端协议视图三者长期对齐。**

比如：

- 要修正 stale in-progress turn
- 要在 watch state 延迟时优先反映真实 running turn
- 要把 rollout 历史和 live current turn 叠起来
- 要避免旧 listener 退出时误删新 listener

这些都不是某个“中心大状态机”一次性解决掉的，而是由一组边界函数持续校正出来的。

这和整个 Codex 架构的风格是一致的：

- 大对象负责主骨架
- 小函数负责边界正确性

---

## 11. 这一章最重要的判断

读完这章，应该稳定这些结论。

### 判断 1：app-server 不拥有 thread 真相
它拥有的是 thread 的控制面和协议投影面。

### 判断 2：listener 是线程事件泵
它不是普通消息转发器，而是 event 与 command 串行化的关键上下文。

### 判断 3：listener generation 是防 stale-task 机制
它确保旧 listener 不会误清理新 listener。

### 判断 4：`apply_bespoke_event_handling` 是协议投影总汇点
它把 raw event 升级为 app-server 语义通知。

### 判断 5：`thread_state_manager` 与 `thread_watch_manager` 是刻意拆开的
前者管内部协调，后者管外部状态投影。

### 判断 6：`active_turn_snapshot` 是 live turn 与恢复历史的桥点
它让 persisted history 能和当前活跃态合并。

### 判断 7：running-thread resume 靠 listener context 保证顺序
不是靠外部随手拼响应。

### 判断 8：pending request replay 是 thread runtime 语义的一部分
不只是 transport retry。

---

## 12. 下一章该接什么

到这里，thread 和 listener 主线已经立住了。接下来最自然的问题是：

> **这些 event 最终是怎样被切成 turn、怎样从 rollout replay 成 history、怎样决定一个 turn 的语义边界？**

也就是说，下一章应该专门讲：

- `ThreadHistoryBuilder`
- `handle_user_message`
- `active_turn_snapshot`
- rollback / compaction / replay 这些 turn 语义问题

也就是下一章的主题：**turn-history 语义层。**

---

## 相关阅读

- 想继续看 turn 怎么从 event stream 被切出来：[[04-turn-history语义层]]
- 想看 pending request / resolve 这条微缺口：[[14-ServerRequestResolved覆盖面与未迁移疑点]]
- 想看这条线最关键的函数和调用顺序：[[11-关键函数索引]]、[[12-调用链索引]]

## 关键文件

- `codex-rs/app-server/src/codex_message_processor.rs`
- `codex-rs/app-server/src/thread_state.rs`
- `codex-rs/app-server/src/thread_status.rs`
- `codex-rs/app-server/src/outgoing_message.rs`
- `codex-rs/app-server/src/bespoke_event_handling.rs`
- `codex-rs/core/src/thread_manager.rs`
- `codex-rs/app-server-protocol/src/protocol/thread_history.rs`
