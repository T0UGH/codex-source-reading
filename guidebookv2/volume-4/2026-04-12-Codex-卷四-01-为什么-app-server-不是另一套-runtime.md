---
title: 为什么 app-server 不是另一套 runtime，而是建立在 core 之上的控制面 facade
date: 2026-04-12
status: draft
tags:
  - Codex
  - guidebook
  - volume-4
  - app-server
  - control-plane
  - runtime
---

# 为什么 app-server 不是另一套 runtime，而是建立在 core 之上的控制面 facade

## 先回答读者最容易问的那个问题

**Codex 里 app-server 看起来非常厚：有 message processor，有 listener，有 thread state，有 watch state，还有一整套对外协议。那为什么不能直接说：它就是 Codex 的另一套 runtime？**

先给结论：

> **不能这样理解。app-server 虽然厚，但它主要不是在重新持有 thread runtime，而是在把 core 里已经成立的 thread runtime，整理成一层可观察、可控制、可恢复连接的控制面。**
>
> 更准确地说：
>
> 1. **runtime owner 在 `core / ThreadManager` 这一侧，不在 app-server；**
> 2. **app-server 主要是在把已经成立的 live thread 语义，暴露成稳定的控制面 facade；**
> 3. **它补出来的重点不是“再跑一遍 runtime”，而是连接、订阅、状态投影、resume、reconnect、request replay 这些控制面能力。**

所以这篇的任务很单纯：

> **先把“谁拥有 runtime，谁暴露控制面”这条边界钉死。**

这也顺手把它和后文分开：

- **第 01 篇回答：为什么 app-server 不是另一套 runtime；**
- **第 02 篇再回答：控制面作为机制到底靠什么站稳；**
- **第 03 / 04 篇再进入 request semantics 的细分边界。**

---

## 先把几个关键词说白

### runtime owner

本文说的 **runtime owner**，可以先理解成：

> **那一层真正创建、持有并推进 live thread 的运行实体。**

如果某层是 runtime owner，它至少要更接近这些事实：

- thread 在哪里出生；
- live thread handle 在哪里被取回；
- resume 时真正恢复的 thread 是谁；
- thread 生命周期最终由谁持有。

从源码关系看，这一侧更接近 `core::ThreadManager`，而不是 app-server。

### facade

本文说的 **facade**，不是“薄薄一层壳”。

这里它更接近：

> **把底层已经存在的复杂能力，整理成更稳定、更适合外界使用的一层总接口。**

所以 facade 可以很厚，也可以承担很多正确性工作；但它承担的重点仍然是**暴露、整理、协调、收口**，而不是重新成为底层事实的 owner。

### control plane

这里的 **control plane**，可以先直白理解成：

> **外界用来观察、控制、恢复连接、继续接回 thread 的那一层正式操作面。**

对 Codex 来说，这层要解决的不是“thread 能不能跑起来”，而是：

- 客户端怎么连上；
- 连上后怎么订阅 thread；
- 断开后怎么 resume；
- 还没解决的 request 怎样 replay；
- 当前 thread 对外应该显示成什么状态。

这正是 app-server 擅长的部分。

---

## 本文只先立住一个总判断：app-server 是控制面，不是平行 runtime

如果从目录体量看，app-server 很容易给人一种直觉：

- 它有自己的状态；
- 它有自己的事件处理；
- 它有自己的 listener；
- 它有自己的 request/response/notification；
- 它还直接面向 TUI、SDK、remote client。

于是很容易得出一个错误结论：

> **既然它这么厚，那它大概就是和 core 平行的一套 runtime。**

但把调用关系和职责边界放在一起看，情况更接近下面这句话：

> **core 负责让 thread 真正活起来，app-server 负责让外界稳定地使用、观察、控制并重新接回这些 thread。**

所以这篇要先稳定 4 个判断：

1. **thread runtime 的 owner 更靠近 `core::ThreadManager`；**
2. **app-server 从一开始就把 `ThreadManager` 当成自己的 runtime 入口；**
3. **app-server 额外补出来的是控制面状态，而不是另一份 thread 真相；**
4. **TUI、SDK、remote client 越来越围绕 app-server 收敛，不是因为它比 core 更底层，而是因为它更适合作为统一控制面。**

---

## 一、为什么说 runtime owner 在 core / ThreadManager 一侧

先看最硬的判断标准：**thread 是在哪里被创建、被取回、被恢复的。**

现有源码链路非常直接。

### 1. app-server 启动时，先造的是 `ThreadManager`

在 `app-server/src/message_processor.rs` 里，app-server 启动时会先构造：

- `AnalyticsEventsClient`
- `ThreadManager::new(...)`
- 然后把这个 `thread_manager` 注入 `CodexMessageProcessor`

这件事很关键。它说明 app-server 并没有自己再定义一套 thread runtime 再把 core 包进去；相反，它从一开始就承认：

> **真正的 thread runtime 入口，应该由 `ThreadManager` 提供。**

### 2. `CodexMessageProcessor` 自己也把 `thread_manager` 当成核心依赖

`CodexMessageProcessor` 里并列存在这些字段：

- `thread_manager`
- `thread_state_manager`
- `thread_watch_manager`
- `outgoing`
- 其他控制面管理对象

这组并列关系本身就很说明问题。

如果 app-server 自己是 runtime owner，那么 `thread_manager` 更像一个被它调用的下层细节。
但现在更像反过来：

- `thread_manager` 负责 thread runtime；
- `thread_state_manager`、`thread_watch_manager` 负责 app-server 自己补出的控制面状态；
- `outgoing` 负责向外发协议消息。

也就是说，app-server 不是把 `ThreadManager` 吞进去，而是围着它再加一层控制面组织。

### 3. thread 的创建、取回、恢复，都直接落到 `ThreadManager`

从 `core/src/thread_manager.rs` 和 app-server 到 core 的桥接路径可以看到：

- 取 thread：`thread_manager.get_thread(thread_id)`
- 起 thread：`thread_manager.start_thread_with_tools_and_service_name(...)`
- resume thread：`thread_manager.resume_thread_with_history(...)`

这三个动作恰好就是 runtime owner 最关键的证据：

- **创建权**；
- **持有权**；
- **恢复入口权**。

如果这些动作都不在 app-server 自己手里，那就很难说它是另一套平行 runtime。

更准确的说法只能是：

> **thread 先在 core 里成立，app-server 再把它包装成可订阅、可展示、可恢复连接的服务端对象。**

---

## 二、app-server 真正补出来的，是控制面状态，不是另一份 thread 真相

很多误读来自一个事实：**app-server 的确也有很多“状态”。**

但关键不在“有没有状态”，而在“这些状态是不是 thread runtime 本体”。

答案是：大部分不是。

### 1. `thread_state_manager` 更像 listener 局部运行态

从 app-server 的结构看，`thread_state_manager` 管的是这一类问题：

- 当前 thread 有没有 listener；
- listener 的 command channel 在哪；
- 当前连接订阅了什么；
- 当前 active turn 的局部整理状态在哪里；
- raw events opt-in 等 thread-local 细节如何保存。

这些都很重要，但它们描述的是：

> **app-server 为了把 thread runtime 稳定投影出去，需要在本地补哪些控制面辅助状态。**

它不是 thread 自身在 core 里的终极真相源。

### 2. `thread_watch_manager` 更像对外状态投影层

`thread_watch_manager` 负责的又是另一类问题：

- 当前 thread 对外显示成 running 还是 idle；
- 有没有挂起的 permission request；
- 有没有挂起的 user input request；
- 有没有 system error；
- 客户端现在应该把这个 thread 看成什么状态。

这层同样不是在“重新运行 thread”，而是在回答：

> **外界此刻应该如何理解这个 thread。**

这就是典型的控制面职责。

### 3. `outgoing` 和协议对象负责的是“怎么暴露”，不是“谁拥有 runtime”

app-server 还持有大量 request / response / notification / replay 相关逻辑。它确实很重，但这些重量主要来自一件事：

> **把底层 runtime 事件整理成客户端真正能稳定消费的协议面。**

因此，app-server 的重量是真实重量，但这份重量更接近：

- 协议投影；
- 连接管理；
- 顺序控制；
- resume/reconnect 正确性；
- 对外状态修正；
- 控制面语义收口。

而不是“第二套 runtime 引擎”。

---

## 三、为什么 listener 很重要，但仍然不意味着 app-server 拥有另一套 runtime

另一个常见误区是：

- app-server 有 listener task；
- listener 持续接收 event；
- listener 里还要串行处理 resume 和 request resolve；
- 那它不就已经像一个运行时主循环了吗？

这里要特别小心。

正确理解是：

> **listener 是 app-server 里的线程事件泵和控制面串行点，不是 thread runtime 的起源地。**

### 1. listener 接的是 core 已经产生出来的 thread event

listener 的存在，说明 app-server 不能只做“拿到消息就发出去”的薄代理。它需要一个稳定上下文，把：

- core 的事件流；
- resume 相关动作；
- request resolve；
- reconnect 后的 replay；

放进同一条顺序线上处理。

所以 listener 很关键，但它关键在**投影与串行化**，不是在**定义 runtime 所属权**。

### 2. `listener_generation` 这类机制，保护的是控制面正确性

在 `app-server/src/thread_state.rs` 里，`ThreadState` 保存了：

- `cancel_tx`
- `listener_generation`
- `listener_command_tx`
- `current_turn_history`
- `listener_thread`

这里尤其能看出 app-server 的工作重心。

例如 `listener_generation` 的作用，是防止：

- 旧 listener 退出时，误清掉新 listener 的状态；
- 连接重建后，新旧 listener 互相踩状态。

这类设计很成熟，但它保护的是：

> **控制面任务切换时不要出现 stale task 误伤。**

它仍然属于 facade 为了稳定暴露 runtime 而承担的复杂性，不等于 app-server 重新拥有了 runtime 本体。

### 3. listener command channel 的意义，是把关键控制动作拉回同一上下文

`thread_state.rs` 里甚至直接写明：`ThreadListenerCommand` 用来在 thread listener 的上下文里执行操作，以保证串行化。

其中包括两类非常说明问题的动作：

- `SendThreadResumeResponse`
- `ResolveServerRequest`

注释写得很直白：

- resume 要在发送历史的同时原子地重新订阅后续更新；
- request resolve 要在 listener 上下文里执行，以保证 resolved 通知和原 request 的顺序正确。

这说明 app-server 在补的是：

> **控制面顺序正确性。**

也就是说，它在保证“如何对外接回去不乱”，而不是“thread 本身由它重新跑起来”。

---

## 四、为什么 resume / reconnect 最能说明 app-server 是控制面 facade

如果只看 `thread/start`，有些人还会觉得：也许 app-server 只是把 core 当底座，但自己也算一半 runtime。

真正把边界拉清楚的，往往是 **resume / reconnect**。

因为这一步最能暴露 app-server 到底在负责什么。

### 1. 历史恢复语义在 core，连接恢复语义在 app-server

从现有调用链看：

- thread 的恢复仍然通过 `thread_manager.resume_thread_with_history(...)` 落到 core；
- app-server 再继续做 listener attach、thread watch upsert、状态修正、连接恢复等动作。

这可以收成一句很稳的判断：

> **历史与 thread 恢复的 owner 在 core，连接与观察面的恢复 owner 在 app-server。**

这是一条非常健康的分工线。

### 2. running thread 的 resume，不是在外面随手拼响应

现有代码里，running thread 的 resume 会被包装成 `SendThreadResumeResponse`，再送回 listener command channel。

这件事说明 resume 的重点不是“拼一份历史给你”，而是：

- 把历史发出去；
- 原子地把连接重新挂回后续事件流；
- 必要时 replay 这个 thread 上尚未解决的 request；
- 保证这些动作和正在流动的 live event 顺序不打架。

这完全是控制面问题。

也正因为这是控制面问题，才更能说明：

> **app-server 的主价值是把一个已经存在的 live thread，稳定地重新暴露给调用方。**

### 3. pending request replay 也说明它在补“可恢复连接语义”

app-server 不是只做 event forwarding。它还会在 resume 或 reconnect 后，把 thread scope 下尚未解决的 request replay 给新的连接。

这类机制说明的不是“网络补发技巧”，而是：

> **Codex 的 thread 语义允许某些未决控制动作跨连接继续存活，app-server 负责把这层存活语义对外维持住。**

这正是 control plane 的典型职责。

---

## 五、为什么 TUI 越来越像跑在 app-server 之上

这也是本篇必须先压住的一层误解。

有些读者看到 TUI 越来越围绕 app-server 收敛，会进一步误判：

- 既然 TUI 都越来越经过 app-server；
- embedded / remote 也都通过 app-server contract；
- 那 app-server 不就是新的系统中心吗？

这一步要分清楚“统一入口”与“runtime owner”不是一回事。

### 1. 统一入口不等于底层 owner

TUI 侧的 `AppServerSession` 已经明显承担统一会话接口角色。
从现有材料可以看到，thread 生命周期和大量 live event handling 已经高度 app-server 化。

这说明的是：

> **app-server 正在成为统一控制面 contract。**

但这不等于：

> **app-server 变成了 thread runtime 的 owner。**

它只是说明，站在客户端这一侧，经过 app-server 更容易得到统一、稳定、可远端化的接口。

### 2. embedded 与 remote 共用 contract，恰好证明它更像控制面

`app-server-client` 明确同时支持：

- in-process
- remote

而且代码里刻意把这两种形态收进同一个 client facade 中；即使 in-process，也尽量保持 app-server 协议模型。

这恰好说明一件很成熟的设计判断：

> **就算物理上同进程，也尽量不要让上层直接咬 core，而是优先经过统一控制面 contract。**

如果 app-server 是另一套 runtime，这种设计会显得绕；
但如果它是 facade，这就非常合理：

- facade 本来就该屏蔽 transport 差异；
- facade 本来就该把 embedded 和 remote 收成同一接口；
- facade 本来就该让上层尽量少依赖 core 内部对象。

### 3. `legacy_core` 的存在，也反过来说明主方向是 app-server contract

`app-server-client` 里保留了 `legacy_core` 模块，注释写得很清楚：

- 新的 TUI 行为应优先使用 app-server protocol；
- `legacy_core` 只是为了在旧的 startup/config 路径逐步迁移时，暂时去掉对 `codex-core` 的直接依赖。

这说明当前最准确的判断不是“还没决定走哪边”，而是：

> **主控制面方向已经是 app-server，只是边缘路径还在迁移。**

而这 again 说明的是它作为控制面 contract 的地位，不是它取代 core 成为 runtime owner。

---

## 六、把这条边界压成一句最稳的话

到这里，可以把本文结论压成一句最值得记住的话：

> **在 Codex 里，thread runtime 的 owner 更靠近 `core / ThreadManager`；app-server 的职责，是把这份已经成立的 live thread runtime，整理成可调用、可观察、可控制、可恢复连接的控制面 facade。**

换成更白一点的话，就是：

- **core 负责把 thread 真正跑起来；**
- **app-server 负责让外界稳定地接上、看见、控制、接回这条 thread。**

所以 app-server 不是“另一套平行 runtime”，而是：

> **建立在 core runtime 之上的正式控制面。**

---

## 本篇应当留下的 4 个稳定判断

### 判断 1：runtime owner 在 core / ThreadManager 一侧

这一点最不能混。thread 的创建、取回、恢复入口都更靠近 `ThreadManager`，不在 app-server 自己手里。

### 判断 2：app-server 主要做的是控制面暴露

它把 live thread 投影成：

- 可连接；
- 可订阅；
- 可观察；
- 可恢复连接；
- 可 replay 未决请求；
- 可统一供 TUI / SDK / remote client 使用。

### 判断 3：app-server 很厚，但厚在 facade 责任上

它之所以厚，不是因为它偷偷重做了 core，而是因为控制面本来就要承担：

- listener 串行化；
- 状态投影；
- 连接管理；
- 协议语义整理；
- reconnect / resume 正确性；
- 各种边界修正。

### 判断 4：卷四不是“app-server 目录导读”，而是“控制面怎样成立”

因此，接下来的卷四正文不会把重心放在 API 表或文件清单，而会继续回答：

- listener、projection、状态修正怎样把控制面撑起来；
- request semantics 为什么不是一层意思；
- TUI 为什么越来越像跑在 app-server 之上。

---

## 下一篇该接什么

本文先把最基础的边界判断压稳：

> **app-server 不是另一套 runtime，而是 core runtime 之上的控制面 facade。**

接下来最自然的问题就不是继续列 app-server 目录，而是：

> **这层控制面到底靠什么站稳？**

所以下一篇应当进入：

- listener task 为什么是控制面的线程事件泵；
- `apply_bespoke_event_handling(...)` 为什么是协议投影主桥；
- `thread_state_manager` 与 `thread_watch_manager` 为什么要拆开；
- 一组状态修正与收口函数如何把对外视图维持稳定。

也就是卷四第 02 篇要回答的问题：

**listener、协议投影与状态修正，是怎么把控制面真正撑起来的。**
