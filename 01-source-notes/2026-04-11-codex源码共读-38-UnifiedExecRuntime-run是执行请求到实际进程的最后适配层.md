---
title: UnifiedExecRuntime::run(...) 是执行请求到实际进程的最后适配层
date: 2026-04-11
tags:
  - Codex
  - 源码共读
  - unified-exec
  - runtime
  - process-spawn
  - sandbox
source_files:
  - /Users/wangguiping/workspace/codex/codex-rs/core/src/tools/runtimes/unified_exec.rs
  - /Users/wangguiping/workspace/codex/codex-rs/core/src/tools/sandboxing.rs
  - /Users/wangguiping/workspace/codex/codex-rs/core/src/unified_exec/process_manager.rs
  - /Users/wangguiping/workspace/codex/codex-rs/core/src/tools/runtimes/shell/zsh_fork_backend.rs
status: done
---

# Codex 源码共读 38：UnifiedExecRuntime::run(...) 是执行请求到实际进程的最后适配层

## 这篇看什么

前面的执行链里已经知道：
- `UnifiedExecHandler::handle(...)` 负责入口装配
- approval / retry / sandbox attempt orchestration 在更上层
- process manager 负责真正 spawn

那中间最关键的一层就是：

> **一份已经通过审批/策略筛过的 `UnifiedExecRequest`，到底是怎么变成实际可运行进程的？**

也就是：
- remote/local 分支什么时候出现
- shell snapshot wrapping 为什么只在本地做
- zsh-fork backend 为什么是机会主义分支
- sandbox attempt 到底是怎么落成 `ExecRequest`

这篇只讲：

- `UnifiedExecRuntime::run(...)`

## 先给主结论

如果先留一句话，我会写：

> `UnifiedExecRuntime::run(...)` 不是决定“能不能执行”的地方，而是 unified_exec 在 approval/sandbox orchestration 之后的最后适配层：它拿到一份已经被上层认可的 `UnifiedExecRequest` 和一个具体 `SandboxAttempt`，再负责把命令按本地/远端环境、shell 类型、zsh-fork backend、proxy env、sandbox transform 等现实约束整理成最终 `ExecRequest`，然后交给 `UnifiedExecProcessManager` 去真正打开 process session。

再压缩一点：

> **它不做政策判断，它做最后一公里的执行适配。**

---

## 第一层：它不是入口，也不是总调度，而是夹在 orchestrator 和 process manager 中间

高层链路其实是：

1. 上层 orchestrator 决定：
   - approval
   - 是否需要 retry
   - 当前 sandbox attempt 是什么
2. `UnifiedExecRuntime::run(...)` 决定：
   - 这次 attempt 下，真实要怎么构造 launch request
3. `UnifiedExecProcessManager` 决定：
   - 本地 PTY / no-stdin pipe / 远端 exec-server 到底怎么实际启动

所以它的准确位置是：

> **execution policy 之后，spawn mechanics 之前。**

这就是它为什么值钱。

---

## 第二层：它第一步先看的是 environment，不是 sandbox

函数最开始就先判断：
- `environment_is_remote`

这说明 unified_exec 的第一现实分叉不是：
- 读写沙箱是什么

而是：
- 这次执行是在本地环境还是远端环境

### 本地
- 可能做 shell snapshot wrapping
- 可能尝试 zsh-fork

### 远端
- 跳过 snapshot wrapping
- 明确不支持 zsh-fork 路径

这个设计特别合理，
因为 remote/local 的差异是更上游的执行现实约束。

---

## 第三层：shell snapshot wrapping 只在本地做，说明它本质是本地 UX 适配

当环境不是 remote 时，函数会尝试：
- `maybe_wrap_shell_lc_with_snapshot(...)`

这一步的意义不是 sandbox，而是：
- 恢复本地 shell state
- 保持 shell 体验一致性

所以这里非常值得注意的一点是：

> **shell snapshot wrapping 是本地 shell UX 逻辑，不是一般执行逻辑。**

这就是为什么它一旦 remote 就直接跳过。

如果把它强行推到 remote，反而会把远端执行搞脏。

---

## 第四层：PowerShell UTF-8 前缀说明这层在做“平台命令规范化”

在 snapshot wrapping 之后，它还会看：
- 当前 shell 是否 PowerShell

如果是，就加 UTF-8 前缀。

这说明 `run(...)` 的责任不只是 sandbox/env，
还包括：

> **把命令先规范化成一个更适合平台执行的形态。**

也就是说，这一层已经明显不只是“把 request 往下传”。

---

## 第五层：zsh-fork backend 是机会主义分支，不是主路径

当 shell mode 是 `ZshFork(...)` 时，函数会先尝试：
- 构造 sandbox command
- 生成 `ExecRequest`
- 交给 `maybe_prepare_unified_exec(...)`

但如果条件不满足，它不会失败，
而是：
- warn
- 再回退到正常 direct execution 路径

这说明 zsh-fork 的设计定位非常明确：

> **可用就吃，不可用就回落，不是强制成功的唯一执行路径。**

这个决定很成熟。

因为 zsh-fork 依赖：
- Unix
- zsh command shape
- 本地环境
- escalation server/session 等额外条件

它天然就不是主干 universal path。

---

## 第六层：真正的 sandbox 落地发生在 `attempt.env_for(...)`

这是这个函数最关键的边界。

它不会自己手写：
- seccomp
- landlock
- bwrap
- seatbelt
- 这些具体东西

而是把：
- command
- cwd
- env
- additional permissions
- options

先组装成 `SandboxCommand` + `ExecOptions`，
再交给：
- `SandboxAttempt::env_for(...)`

也就是说：

> **`run(...)` 负责提供结构化输入，sandbox manager 负责把它变成真实 `ExecRequest`。**

这让 sandbox retry / unsandboxed second attempt 可以在上层 orchestration 中替换 `SandboxAttempt`，
而不用改这层代码。

这就是很好的分层。

---

## 第七层：它对 zsh-fork 和 normal path 都会先 build sandbox command，说明它真正吃的是“统一执行请求”

无论最后走：
- zsh-fork
- 还是 direct execution

都要先经过：
- `build_sandbox_command(...)`
- `attempt.env_for(...)`

这说明 `run(...)` 并不是在两个完全不同的系统之间切换，
而是在：

> **统一执行请求模型下，选择不同的 spawn backend。**

这是更高级的抽象。

而不是 if/else 下两套完全独立实现。

---

## 第八层：remote path 的真正含义是“交给远端机上的本地 exec backend”

normal path 最后会去：
- `manager.open_session_with_exec_env(...)`

如果 environment 是 remote：
- process manager 会走远端 exec backend
- 本地 inherited FD 这种东西也不支持

所以这里要特别澄清：

> **remote 不是这层自己变成远端 RPC，而是最终交给 process manager 通过远端 environment/backend 去启动。**

这和 `exec-server` 那篇的判断是完全对上的。

---

## 第九层：它把 sandbox denied 保留下来，说明这层明确知道 orchestrator 还要继续处理

spawn 出错时，不是所有错误都一样处理。

如果是：
- `UnifiedExecError::SandboxDenied`

它会转成：
- `ToolError::Codex(SandboxErr::Denied(...))`

而不是简单字符串拒绝。

这说明 `run(...)` 很明确地在给上层 orchestrator 留一个结构化信号：

> **这是 sandbox deny，不是一般执行失败，你可以继续做 approval/retry/escalation 逻辑。**

反过来说，其他错误会被压成普通 rejection，
意味着它们不属于同一类“可自动升级重试”的语义。

这也是很成熟的层间契约。

---

## 第十层：这个函数的核心不是“执行”，而是“把一个已批准的执行请求翻译成一个具体 launch 方案”

我觉得这是最该立住的点。

如果你把它看成 execution function，会觉得它逻辑很杂：
- shell
- env
- proxy
- zsh-fork
- sandbox
- errors
- remote/local

但如果你把它看成：

> **已批准请求 → 具体 launch 方案** 的适配层

它就非常清楚了：
- 上层负责能不能做
- 下层负责怎么真正 spawn
- 它负责中间这段现实世界适配

这就是它最准确的角色定义。

---

## 一句角色定义

如果按源码共读风格压一句话，我会这么写：

> **`UnifiedExecRuntime::run(...)` 是 unified_exec 子系统里把“抽象执行请求”压缩成“当前环境下可实际启动的具体进程方案”的最后适配层。**

---

## 为什么这层设计是对的

### 1. 把 policy 和 spawn 细节隔开
上层 orchestrator 可以专注审批和 retry，下层 manager 专注真正启动。

### 2. 把 snapshot wrapping 限定在本地环境
防止本地 UX 逻辑污染 remote 执行语义。

### 3. zsh-fork 设计成机会主义 fallback-friendly backend
不会让配置项把主链搞脆弱。

### 4. 保留 structured sandbox denial
这让后续重试逻辑有明确机器可读信号。

---

## 继续细拆的话，下一篇最自然的点

最自然继续拆的是：

1. `UnifiedExecProcessManager::open_session_with_exec_env(...)`
2. `ToolOrchestrator::run(...)`

这两个加起来，基本就能把 unified_exec 的最后两层彻底吃透。
