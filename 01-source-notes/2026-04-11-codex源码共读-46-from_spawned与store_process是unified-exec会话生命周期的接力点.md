---
title: from_spawned(...) 与 store_process(...) 是 unified-exec 会话生命周期的接力点
date: 2026-04-11
tags:
  - Codex
  - 源码共读
  - unified-exec
  - process
  - lifecycle
source_files:
  - /Users/wangguiping/workspace/codex/codex-rs/core/src/unified_exec/process.rs
  - /Users/wangguiping/workspace/codex/codex-rs/core/src/unified_exec/process_manager.rs
  - /Users/wangguiping/workspace/codex/codex-rs/core/src/unified_exec/async_watcher.rs
status: done
---

# Codex 源码共读 46：from_spawned(...) 与 store_process(...) 是 unified-exec 会话生命周期的接力点

## 这篇看什么

前一篇已经把 `open_session_with_exec_env(...)` 立住了：
- 它是 spawn 边界适配器
- local/remote 差异在这里被压缩
- 最终返回 `UnifiedExecProcess`

但再往下一层，其实最关键的问题是：

> **一个真正启动起来的进程，什么时候变成 unified-exec 世界里的“会话对象”？什么时候又变成 manager 里可复用的后台 session？**

这篇把两个点放在一起讲：

- `UnifiedExecProcess::from_spawned(...)`
- `store_process(...)`

因为它们其实是一组接力动作。

## 先给主结论

如果先留一句话，我会写：

> `from_spawned(...)` 和 `store_process(...)` 其实分别站在 unified-exec 生命周期的两个相邻边界上：前者负责把一个刚刚 local spawn 成功的底层进程立刻包装成“已带输出缓冲、退出状态、cancellation token 和 early-exit 观察能力”的 `UnifiedExecProcess`；后者则负责决定这个 process session 是否值得被纳入 manager 的长期管理世界——也就是写入 store、参与 LRU/pruning、挂上 exit watcher、拥有后续 `write_stdin/poll` 可见性。前者让进程从 raw spawn 进入 session 语义，后者让 session 从 request-local 对象进入 manager-local 生命周期。

再压缩一点：

> **`from_spawned(...)` 是 session 化，`store_process(...)` 是托管化。**

这个区分我觉得非常关键。

---

## 第一层：`from_spawned(...)` 不是简单 constructor，而是“带 watcher 的 process session 装配器”

它拿到的输入是：
- `SpawnedPty`
- `sandbox_type`
- `spawn_lifecycle`

然后马上做几件关键事：
1. 解构出 process handle / stdout_rx / stderr_rx / exit_rx
2. 把 stdout/stderr merge
3. 建 `UnifiedExecProcess`
4. 起 output task
5. 尝试做 immediate exit probe
6. 再给一个短 grace period 看会不会很快退出
7. 如果仍存活，才起 background exit watcher

这说明它不是 `new(spawned)` 这种静态 constructor，
而是：

> **spawn 之后第一时间把 raw process 装进一个完整 session 外壳。**

所以它的真正工作是 session 化，而不是简单封装。

---

## 第二层：150ms early-exit grace period 说明这层在主动区分“短命失败”和“可托管 session”

`from_spawned(...)` 的一个非常值的点是：
- 它不会一 spawn 成功就立刻认为这是个长期 session
- 而是先给一个 150ms 窗口
- 看它是不是马上就 exit 了

为什么这很重要？

因为很多失败根本不是“长期运行后失败”，而是：
- sandbox deny
- 命令本身立刻报错
- 启动即退出

如果没有这层 early classification，
上层就会把一堆短命失败也当作正常 session 存进 manager store，
后面一堆清理都会变乱。

所以这里的真实作用是：

> **在会话托管之前，先做一轮“它到底像不像一个活会话”的判断。**

---

## 第三层：`from_spawned(...)` 的 cancellation token 不是附属品，而是后续 exit watcher 的主时钟

它在 process session 里会准备：
- output buffer
- output notify
- output closed
- cancellation token
- output_drained
- broadcast channel
- watch state

这里最关键的是：
- cancellation token

因为后面的：
- `start_streaming_output(...)`
- `spawn_exit_watcher(...)`

都要依赖它。

所以在 unified-exec 里，一个 process session 真正被认为“能参与完整生命周期”，
并不是因为有 pid，而是因为已经拥有：

> **统一的 session-level lifecycle signals。**

这也是为什么我说它是在做 session 化。

---

## 第四层：`store_process(...)` 不是普通 hashmap insert，而是把会话纳入 manager 世界

这一点很容易被低估。

`store_process(...)` 会：
- 包出 `ProcessEntry`
  - process Arc
  - call_id
  - process_id
  - command
  - tty
  - network approval id
  - weak session
  - last_used
- 先做 prune
- 再插入 store
- 锁外清理被 prune 掉的 process
- 接近上限时给模型 warning
- 起 `spawn_exit_watcher(...)`

这说明它在做的已经不是“保存引用”，而是：

> **把这个 process session 纳入一个长期可管理、可复用、可清理的资源池。**

所以它的角色是托管化，不只是登记。

---

## 第五层：为什么 `from_spawned(...)` 和 `store_process(...)` 要分开

这是理解 unified-exec 生命周期分层最重要的一点。

如果把两者混在一起，代码会很难看清：
- 哪一步是把 raw spawn 包成 session
- 哪一步是决定要不要长期管理

现在拆开之后，语义非常清楚：

### `from_spawned(...)`
回答：
- 这个底层进程是否已经足够像一个 unified-exec session？

### `store_process(...)`
回答：
- 这个 session 是否值得被 manager 长期追踪？

这就是一个很干净的生命周期分层。

---

## 第六层：`store_process(...)` 真正引入的是资源池语义，而不是单会话语义

它一旦入 store，就会开始受这些规则支配：
- `MAX_UNIFIED_EXEC_PROCESSES`
- LRU / recent protection
- exited process 优先 prune
- warning threshold
- network approval cleanup
- background exit watcher

这说明 process 一旦进入 store，
它的身份就不再只是“某次请求的子产物”，
而变成：

> **整个 unified-exec manager 要共同管理的一项资源。**

这就是资源池语义。

---

## 第七层：pruning 策略说明 unified-exec 已经明确是“有限池”而不是“无限开会话”

`store_process(...)` 里最成熟的点之一就是 pruning：
- 最多 64
- 最近 8 个强保护
- 先 prune 退出的
- 不够再 prune 更久没用的

这说明系统从架构上已经承认：

> **interactive/background process session 是昂贵资源，必须有限管理。**

而不是让 agent 无限制开会话。

这比很多 demo 级终端系统成熟得多。

---

## 第八层：为什么 `store_process(...)` 一定要在 request 还没结束时就做

外层 `exec_command(...)` 会先：
- `start_streaming_output(...)`
- 然后很快 `store_process(...)`

注释里明确说了：
- 要先存，不然中断当前 turn 时最后一个 `Arc` 可能掉光，进程直接死掉

这说明 `store_process(...)` 其实还承担了：

> **把 process 生命周期从 request-local 迁移到 manager-local，避免 request 一结束就把活进程一起带死。**

这点非常关键。

如果没有这一层，interactive/background session 根本立不起来。

---

## 第九层：把 `from_spawned(...)` 和 `store_process(...)` 连起来看，能看到 unified-exec 的完整生命周期接力

我会把它压成这样：

### 接力 1：spawn → session
- `from_spawned(...)`
- 解决输出、退出、sandbox-denial、cancellation 基础设施

### 接力 2：session → managed resource
- `store_process(...)`
- 解决登记、容量、复用、清理、最终 end-event watcher

这两步连起来，才构成完整 unified-exec 生命周期。

所以我会说：

> **前者做 session 化，后者做托管化。**

---

## 一句角色定义

如果按源码共读风格压一句话，我会这么写：

> **`from_spawned(...)` 把 raw spawn 变成 unified-exec session，`store_process(...)` 再把这个 session 变成 manager 可长期管理的后台资源；它们是一条连续生命周期里的两次身份升级。**

---

## 为什么这层设计是对的

### 1. 把 session 化和托管化分开
职责很清楚。

### 2. early-exit classification 放在 session 化阶段
避免短命失败混进 store。

### 3. resource-pool policy 放在托管化阶段
避免 spawn 逻辑被容量策略污染。

### 4. 先 store，再让 request 继续推进
保证 interactive/background session 不会被 request 生命周期误杀。

---

## 继续细拆的话，下一篇最自然的点

最自然继续拆的是：

1. `spawn_exit_watcher(...)`
2. `start_streaming_output(...)`
3. `refresh_process_state(...)`

这三篇会把 unified-exec 后半段 watcher / end-event / store 清退的逻辑彻底讲透。
