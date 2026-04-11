---
title: open_session_with_exec_env(...) 是 spawn 边界适配器
date: 2026-04-11
tags:
  - Codex
  - 源码共读
  - unified-exec
  - process-manager
  - spawn
  - exec-server
source_files:
  - /Users/wangguiping/workspace/codex/codex-rs/core/src/unified_exec/process_manager.rs
  - /Users/wangguiping/workspace/codex/codex-rs/core/src/unified_exec/process.rs
  - /Users/wangguiping/workspace/codex/codex-rs/core/src/unified_exec/async_watcher.rs
status: done
---

# Codex 源码共读 42：open_session_with_exec_env(...) 是 spawn 边界适配器

## 这篇看什么

前一篇已经把 `UnifiedExecRuntime::run(...)` 立住了：
- 它负责最后一公里的 launch 方案适配
- policy/orchestration 在上层
- 实际 spawn mechanics 在下层 manager

那真正最自然的下一刀就是：

> **`open_session_with_exec_env(...)` 到底是怎么把一个已准备好的 `ExecRequest` 变成真正的 process session 的？**

也就是：
- remote/local 分支怎么分
- inherited FDs 为什么只在本地支持
- `SpawnLifecycleHandle` 真正在哪个点起作用
- 这个函数自己负责哪些生命周期动作，哪些交给外层/后层

这篇只讲：

- `UnifiedExecProcessManager::open_session_with_exec_env(...)`

## 先给主结论

如果先留一句话，我会写：

> `open_session_with_exec_env(...)` 不是 unified_exec 的总控函数，而是 process manager 里的 spawn 边界适配器：它拿到一个已经准备好的 `ExecRequest`、`tty` 选择、spawn lifecycle 和 execution environment 后，先统一校验 command line，再按 `environment.is_remote()` 分成远端 exec-server 路径和本地 PTY/pipe 路径，处理 inherited FDs、`after_spawn()` 生命周期回调、spawn error 归一化，最后返回一个已经带输出/退出监控能力的 `UnifiedExecProcess`。它不负责 policy，也不负责把 process 存进 manager store；它只负责把“可执行请求”跨过 spawn 边界变成“已启动 session”。

再压缩一点：

> **它不是 exec runtime 总控，而是 local/remote spawn 的边界转换层。**

---

## 第一层：它先检查 command line 是否存在，说明这是一条统一 spawn invariant

函数一开始先做：
- `request.command.split_first()`
- 没有 program 就报 `MissingCommandLine`

这个动作看着普通，但很说明问题。

因为不管：
- 本地 spawn
- 还是远端 exec-server start

都共用同一个 invariant：

> **统一执行请求必须至少有一个 command element。**

也就是说，这一层在 local/remote 分支之前，先统一了 spawn 前提。

---

## 第二层：`SpawnLifecycleHandle` 的真正入口就在这里

函数接着会立刻读：
- `spawn_lifecycle.inherited_fds()`

这说明：
- lifecycle 不是 attach 在某个更外层抽象上的装饰物
- 它在 spawn 边界这一步就已经开始影响真实行为了

而且影响方式很具体：
- inherited FDs 要不要跟着 spawn 过去

这就说明 `SpawnLifecycleHandle` 的角色非常像：

> **spawn 边界上的资源携带/释放协议。**

不是普通回调。

---

## 第三层：remote 分支最关键的判断不是怎么连 exec-server，而是“不能带 inherited FDs”

一旦 `environment.is_remote()`，函数先做的不是远程连接，
而是：
- 如果 `inherited_fds` 非空，直接 reject

这一步特别值。

它说明作者非常明确：

> **远端 exec-server 路径和本地 Unix 进程继承 FD 模型不兼容。**

而且这个不兼容不是尝试兼容，而是硬边界。

这很对。

否则如果偷偷丢掉 FDs 或假装支持，问题会更难查。

---

## 第四层：remote path 真正做的是“把 manager 侧 process id 和 request 重新编码成 exec-server start 参数”

remote 分支会把：
- `process_id`
- `argv`
- `cwd`
- `env`
- `tty`
- `arg0`

重新打包给：
- `environment.get_exec_backend().start(...)`

这说明在 manager 看来：
- remote 不是自己特殊控制很多逻辑
- 而是通过 execution environment 的 backend 接口去发起远程启动

所以这里再次坐实一个判断：

> **真正抽象 local/remote 差异的不是 process manager，而是 environment/backend。**

manager 只是利用这个抽象在 spawn 边界做分流。

---

## 第五层：`from_remote_started(...)` / `from_spawned(...)` 说明它返回的不是裸句柄，而是一个“已接好 watcher 的 process session”

这个点非常关键。

函数最后不是返回：
- raw remote handle
- raw local child

而是返回：
- `UnifiedExecProcess`

并且：
- remote path → `from_remote_started(...)`
- local path → `from_spawned(...)`

这说明 spawn 不是结束点，
而是要立刻进入一个更高层语义：

> **已启动、可读输出、可跟踪退出、可继续交互的 process session。**

所以这个函数跨过去的边界不是：
- request → raw spawn

而是：
- request → managed session object

这层抽象非常重要。

---

## 第六层：本地分支先选 PTY 还是 pipe，说明 interactivity 是 spawn 边界上决定的，不是事后附加的

如果本地：
- `tty == true` → PTY
- `tty == false` → no-stdin pipe

这说明 interactive/non-interactive 不是同一条 spawn 路径上的小差异，
而是：

> **spawn 方式本身就不同。**

这也和后面 `write_stdin(...)` 的约束对得上：
- non-tty session 本来就没有 stdin 交互语义

所以 interactivity 是 spawn-time reality，不是 session 后面补出来的能力。

---

## 第七层：`after_spawn()` 放在成功本地 spawn 之后，说明 lifecycle hook 的真正边界是“资源可以安全释放的时刻”

本地分支在成功 spawn 之后会：
- `spawn_lifecycle.after_spawn()`

注意这个调用时机很讲究：
- spawn 前不能调
- spawn 失败也不能调
- 一定要在 child 真正起来之后再调

这说明作者对 lifecycle hook 的理解不是：
- “给你一个 spawn 成功通知”

而是：
- **父进程侧某些资源现在终于可以安全切换/释放了。**

这和 inherited FDs 的语义刚好对上。

所以这里真正封装的是：

> **spawn 边界的资源生死线。**

---

## 第八层：这个函数自己不负责 store / begin / end，说明它刻意只守 spawn 边界

这一点特别重要。

它不会：
- emit begin event
- store process entry
- spawn exit watcher
- emit end event
- release process id

这些都在外层 `exec_command(...)` 和后续 watcher 逻辑里完成。

说明作者是明确把这层函数限制在：

> **“把它真正启动起来”**

而不是：

> **“把整个 unified_exec 生命周期全包了”**

这是非常好的分层。

否则 manager 这一层会膨胀得很快。

---

## 第九层：它的真正价值是把 local/remote 差异压缩成同一个 `UnifiedExecProcess` 语义

虽然内部路径差异很大：
- remote 走 exec-server
- local 走 PTY/pipe
- remote 不支持 inherited FDs
- local 支持 lifecycle hook

但最后外面拿到的都是：
- `UnifiedExecProcess`

这说明这个函数真正完成的是：

> **把不同底层 spawn 世界，压缩成统一的 session 语义。**

这也是为什么上层 unified exec runtime 可以相对干净。

---

## 一句角色定义

如果按源码共读风格压一句话，我会这么写：

> **`open_session_with_exec_env(...)` 是 unified_exec 里最核心的 spawn 边界适配器：它把一个已准备好的执行请求跨过 local/remote 真实执行差异，转换成一个统一的、已接好 watcher 的 process session。**

---

## 为什么这层设计是对的

### 1. 把 local/remote 差异压在 spawn 边界
上层不需要满世界知道 exec-server 细节。

### 2. 把 lifecycle hook 也压在 spawn 边界
资源携带/释放时机最清楚。

### 3. 只做 spawn，不做整段生命周期
函数职责边界很稳。

### 4. 统一返回 `UnifiedExecProcess`
大幅减轻上层复杂度。

---

## 继续细拆的话，下一篇最自然的点

最自然继续拆的是：

1. `UnifiedExecProcess::from_spawned(...)`
2. `UnifiedExecProcess::from_remote_started(...)`
3. `store_process(...)`

这三篇会把：
- watcher / output / exit state
- live session 管理

彻底讲透。
