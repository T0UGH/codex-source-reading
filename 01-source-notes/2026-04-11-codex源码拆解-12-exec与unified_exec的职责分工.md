# Codex 源码拆解 12：exec 与 unified_exec 的职责分工

## 这篇看什么

回答：**`core/src/exec.rs`、tool handler 里的 `unified_exec`、runtime 里的 `unified_exec` 到底谁管什么？**

## 先给结论

这三层不是重复造轮子，而是刻意分层：

1. `core/src/exec.rs`
   - 更底层的 **进程执行与 sandbox request 构建层**
   - 偏 execution primitive
2. `core/src/tools/handlers/unified_exec.rs`
   - **模型工具入口层**
   - 负责解析参数、权限校验、implicit skill telemetry、apply_patch 拦截、调用 manager
3. `core/src/tools/runtimes/unified_exec.rs`
   - **approval + sandbox orchestration runtime**
   - 负责把统一 exec 请求跑过审批/沙箱策略后交给 process manager

一句话：

> `exec.rs` 是底盘，`handler` 是 tool 入口，`runtime/unified_exec.rs` 是有审批语义的执行编排层。

## 关键代码路径

- `codex-rs/core/src/exec.rs`
- `codex-rs/core/src/tools/handlers/unified_exec.rs`
- `codex-rs/core/src/tools/runtimes/unified_exec.rs`
- `codex-rs/core/src/state/service.rs`

## 证据 1：`exec.rs` 明显是 execution primitive

`exec.rs` 里定义了：

- `ExecParams`
- `ExecExpiration`
- `ExecCapturePolicy`
- `process_exec_tool_call(...)`
- `build_exec_request(...)`

它做的核心事情是：

- 选择 sandbox type
- 组装 `ExecRequest`
- 统一走 `crate::sandboxing::execute_env(exec_req, ...)`

注释也写得很直白：

> Route through the sandboxing module for a single, unified execution path.

所以这里的“unified”更像 **执行底座统一**，不是产品语义层的 unified_exec tool。

## 证据 2：tool handler 是模型调用入口，不是进程底盘

`tools/handlers/unified_exec.rs` 一眼就能看出是工具层：

- `ExecCommandArgs` / `WriteStdinArgs`
- `ToolHandler for UnifiedExecHandler`
- `is_mutating()`
- `pre_tool_use_payload()` / `post_tool_use_payload()`
- `handle()`

它承担的是：

- 解析 tool arguments
- 判断 command 是否已知安全
- 处理 approval policy 与 escalated permissions 约束
- 触发 `maybe_emit_implicit_skill_invocation(...)`
- 特判 `apply_patch`
- 向 `session.services.unified_exec_manager` 发起 exec / write_stdin

这层明显不是底层 exec API，而是：

> “模型说要执行命令” 时的入口适配器。

## 证据 3：SessionServices 明确把 unified exec 当共享会话服务

`state/service.rs` 里：

- `pub(crate) unified_exec_manager: UnifiedExecProcessManager`

这说明 unified exec 在当前架构里不是一次性 helper，而是 **session 级共享过程管理器**。

也解释了为什么它要支持：

- process id 分配
- session 复用
- stdin 持续写入
- 背景/交互进程生命周期

这已经不是传统“一次 exec 然后收输出”的模型了。

## 证据 4：runtime/unified_exec.rs 是审批与沙箱编排层

文件头注释很明确：

> Handles approval + sandbox orchestration for unified exec requests, delegating to the process manager to spawn PTYs once an ExecRequest is prepared.

它定义：

- `UnifiedExecRequest`
- `UnifiedExecApprovalKey`
- `UnifiedExecRuntime`

并实现：

- `Sandboxable`
- `Approvable`
- `ToolRuntime<UnifiedExecRequest, UnifiedExecProcess>`

这说明这层的关键价值是：

- 生成 approval cache key
- 触发 guardian / command approval
- 决定 first attempt sandbox mode
- 构造 network approval spec
- 在最终 `run()` 里把 request 交给 `UnifiedExecProcessManager`

所以它不是“参数解析层”，也不是“spawn syscall 层”，而是中间那层 **带策略语义的 runtime adapter**。

## 一个更准确的分层图

### 第 1 层：tool surface
`UnifiedExecHandler`
- 面向模型 function/tool call
- 参数解析、权限前置、patch 特判、telemetry

### 第 2 层：policy runtime
`tools/runtimes/unified_exec.rs`
- approval
- sandbox attempt
- network approval
- command wrapping
- 交给 manager 打开会话

### 第 3 层：process/session manager
`UnifiedExecProcessManager`
- process id
- open session
- write stdin
- terminate/cleanup

### 第 4 层：execution primitive
`exec.rs` + `sandboxing`
- `ExecRequest`
- sandbox transform
- execute_env

## 设计判断

### 1. unified_exec 不是一个函数，而是一整套产品级执行子系统

这是当前最重要的认识。

它覆盖：
- model tool 入口
- approval
- policy
- PTY/session lifecycle
- process manager
- sandbox execution primitive

### 2. `exec.rs` 仍然是底层通用执行底盘

不能因为有 `unified_exec` 就说 `exec.rs` 过时。

更准确地说：

- `exec.rs` 提供 execution primitive
- `unified_exec` 提供 agent-facing / sessionful exec product

### 3. `exec` vs `unified_exec` 的差异，本质是“单次执行” vs “会话化执行”

`exec.rs` 更像 command execution substrate。

`unified_exec` 更像：
- 带审批
- 带交互 stdin
- 带 process tracking
- 带 UI/TUI 状态

的执行产品。

## 当前边界判断

- `exec.rs`：底层执行与 sandbox request 构造
- `UnifiedExecHandler`：tool 入口适配层
- `tools/runtimes/unified_exec.rs`：审批/沙箱编排层
- `UnifiedExecProcessManager`：会话化进程管理层
- `sandboxing`：最终执行环境实现

## 一个经验性判断

如果后面还要继续拆，优先别再问“哪个才是真的 exec”。

更应该问：

1. 哪层面向模型
2. 哪层面向审批策略
3. 哪层面向 process/session 生命周期
4. 哪层面向真实 OS 执行

这样就不会被命名绕进去。
