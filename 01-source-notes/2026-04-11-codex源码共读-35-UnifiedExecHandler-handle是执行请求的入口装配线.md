---
title: UnifiedExecHandler::handle(...) 是执行请求的入口装配线
date: 2026-04-11
tags:
  - Codex
  - 源码共读
  - unified-exec
  - tool-runtime
  - permissions
  - execution
source_files:
  - /Users/wangguiping/workspace/codex/codex-rs/core/src/tools/handlers/unified_exec.rs
status: done
---

# Codex 源码共读 35：UnifiedExecHandler::handle(...) 是执行请求的入口装配线

## 这篇看什么

前面的执行链架构稿已经收住了：
- `exec.rs` 是底层执行 primitive
- `unified_exec` 是会话化执行子系统
- process manager / runtime / handler 是分层的

但如果按更细的源码共读视角，真正关键的问题是：

> 模型发起一次 `exec_command`，到底是怎么被接进整个 unified_exec 子系统的？

也就是：
- 为什么这个 handler 不是简单 parse 一下参数就调 manager
- permission / sandbox / sticky turn permissions 是怎么插进来的
- `apply_patch` 为什么在这里被 special-case 拦截
- `write_stdin` 为什么也放在同一个 handler 里

这篇只讲一个函数：

- `UnifiedExecHandler::handle(...)`

## 先给主结论

如果先留一句话，我会写：

> `UnifiedExecHandler::handle(...)` 不是执行器本身，而是 unified_exec 子系统的入口装配线：它把模型侧 function call payload 先翻译成 runtime 可用的执行请求，再把隐式 skill、workdir 解析、shell 展开、sticky permissions、额外权限合法性、`apply_patch` 短路、TTY telemetry、process id 分配等一系列前置动作都接好，最后才把真正的 `ExecCommandRequest` 交给 `UnifiedExecProcessManager`。

再压缩一点：

> **它不是“执行命令”的地方，而是“把命令装配成可执行请求”的地方。**

---

## 第一层：它先决定自己是不是在正确上下文里

函数一开始先做两步硬检查：

1. payload 必须是 `ToolPayload::Function`
2. `turn.environment` 必须存在

如果没有 environment，就直接报：
- `unified exec is unavailable in this session`

这说明在 Codex 里，unified exec 不是任何 session 都默认存在。

也就是说，这个 handler 最先确认的不是 command，
而是：

> **当前 session 有没有执行环境资格。**

这也和前面 `EnvironmentManager/Environment` 的系统级判断是对上的。

---

## 第二层：它不是直接拿 arguments 执行，而是先建 `UnifiedExecContext`

接下来它会把：
- `session`
- `turn`
- `call_id`

装进：
- `UnifiedExecContext`

这说明后续不是简单的函数式执行，
而是围绕一个运行时 context 往下推进。

这和你之前 CC 那些 `runAgent/queryLoop` 的感觉很像：
- 不是只把参数传下去
- 而是先构造一个后面所有动作都依赖的执行上下文

---

## 第三层：`exec_command` 分支真正做的是“参数翻译 + 执行请求装配”

当 `tool_name == "exec_command"` 时，它会依次做：

1. resolve workdir base path
2. parse `ExecCommandArgs`
3. 计算逻辑上的 `workdir`
4. `maybe_emit_implicit_skill_invocation(...)`
5. 向 manager 申请 `process_id`
6. `get_command(...)` 做 shell 展开
7. 处理 permission / sandbox / additional permissions
8. special-case `apply_patch`
9. 记 TTY metric
10. 真正调用 `manager.exec_command(...)`

这条链说明它根本不是一个薄 handler。

它更像：

> **tool payload 到 runtime request 的装配层。**

---

## 第四层：隐式 skill invocation 插在这里，非常合理

在真正 allocate process / build command 之前，它会：
- `maybe_emit_implicit_skill_invocation(...)`

说明 unified_exec 不是一个纯“命令执行器”，
而是运行在更大的 skill/runtime 语境里。

也就是说，这里已经把“命令是什么”与“这个命令命中了什么隐式技能经验/统计”接起来了。

这一步很值，因为它证明：

> **skill 不是提示词孤岛，而是会在 tool 入口层留下痕迹的能力层。**

---

## 第五层：`process_id` 不是执行后才知道的，而是执行前先申请的

这个细节很关键。

它先：
- `manager.allocate_process_id().await`

再继续往下装配请求。

说明在 unified_exec 的设计里，process id 不是底层 spawn 后再被动拿回来，
而是上层 runtime 先预留一个 process slot。

这有几个直接好处：
- 后续 approval / UI / stdin 续写可以稳定引用这个 id
- request 失败时也能明确 release
- process 生命周期在 runtime 层更可控

这就不是“shell 一次性命令调用”的心智了，
而是：

> **会话化 process runtime** 的心智。

---

## 第六层：permission 不是一个布尔值，而是一整段前置装配逻辑

这是这个函数最像你 Claude Code 权限共读稿的地方。

它会做：

### 1. 先看 feature flag
- `ExecPermissionApprovals`
- `RequestPermissionsTool`

### 2. 应用 sticky turn permissions
- `apply_granted_turn_permissions(...)`

### 3. 判断当前 approval policy 下，是否允许请求更高权限
如果 approval policy 不是 `OnRequest`，
而当前命令又要求 escalated permissions，
那就直接 reject。

### 4. 归一化 additional permissions
通过：
- `implicit_granted_permissions(...)`
- `normalize_and_validate_additional_permissions(...)`

最终把：
- requested permissions
- sticky granted permissions
- preapproved 状态
- 当前 cwd

综合成真正可下发的权限请求。

这说明在 unified_exec 里，权限绝不是个边缘逻辑，
而是这条主链的正式前置阶段。

一句话：

> **执行请求先经过权限装配，才能成为真正的执行请求。**

---

## 第七层：`apply_patch` 被专门短路，说明 unified_exec 是兼容层，不是唯一执行面

这里有个很值的 special-case：
- `intercept_apply_patch(...)`

如果命中的确是 patch 流，
它会直接返回 patch 输出，
而不会继续走 normal exec_command 流。

这说明 `exec_command` 这个入口在模型视角下可能承接多种命令文本，
但系统并不傻到把 `apply_patch` 真的当普通 shell command 去跑。

所以这个 handler 还承担了：

> **“识别这其实不是一般 shell 执行，而是该走专门 patch 语义”的兼容与分流责任。**

这正是入口装配层该做的事。

---

## 第八层：真正下发给 manager 的已经不是原始参数，而是被“清洗过”的 `ExecCommandRequest`

最终交给 manager 的东西里，已经包含：
- `command`
- `process_id`
- `yield_time_ms`
- `max_output_tokens`
- `workdir`
- `network`
- `tty`
- `sandbox_permissions`
- `additional_permissions`
- `additional_permissions_preapproved`
- `justification`
- `prefix_rule`

注意这里的 `sandbox_permissions` 和 `additional_permissions`，
已经不是原始用户请求，而是：
- sticky 权限合并后
- approval policy 过滤后
- cwd 校验后
- normalized 后

的结果。

所以 manager 接到的并不是“带噪音的模型输入”，
而是：

> **runtime-ready request**

这就是 handler 存在的真正意义。

---

## 第九层：`write_stdin` 也放在这个 handler 里，说明 unified_exec 的产品心智是“会话化终端”

另一个分支是：
- `write_stdin`

它会：
- parse `WriteStdinArgs`
- `manager.write_stdin(...)`
- 再主动发一个 `TerminalInteractionEvent`

这说明 unified_exec 设计的不是：
- 一次命令执行完就结束

而是：
- 进程还能继续交互
- stdin 还能被续写
- 交互还会进入事件流

也就是说，这里再次证明：

> **unified_exec 是会话化 terminal runtime，不是单次 shell wrapper。**

---

## 第十层：这个函数最准确的角色，不是 execution，而是 pre-execution composition

我觉得这是最该立住的一点。

如果你把它理解成“执行函数”，会觉得它做得太杂。

但如果你把它理解成：

> **一次统一执行请求在进入 runtime 前的装配与校验层**

那它的所有动作都很合理：
- 解析 payload
- 校验 session 环境
- 绑定 context
- workdir/base path 解析
- 隐式 skill 统计
- process id 分配
- shell 展开
- 权限融合
- patch 分流
- telemetry
- 交 manager

这就是典型的入口装配线。

---

## 一句角色定义

如果按你之前 CC 笔记那种风格，我会这么压：

> **`UnifiedExecHandler::handle(...)` 不是命令执行器本身，而是 Codex 里把“模型说要执行一段命令”翻译成“runtime 可以接受的一份会话化执行请求”的入口装配线。**

---

## 为什么这层设计是对的

### 1. 把权限和 shell 展开放在入口层
避免下层 manager/runtimes 被模型脏输入污染。

### 2. 先分配 process id，再执行
这让 unified_exec 可以天然支持续写 stdin、事件追踪、UI 绑定。

### 3. patch 兼容放在这里
这让 `exec_command` 入口能兼容模型的命令文本习惯，又不破坏系统内部专门语义。

### 4. `write_stdin` 和 `exec_command` 同 handler
这进一步说明 unified_exec 是统一的 terminal interaction surface。

---

## 继续细拆的话，下一篇最自然的点

如果继续往下拆，最自然的是：

1. `UnifiedExecRuntime::run(...)`
   - 这层怎么把 approval/sandbox orchestration 接起来
2. `UnifiedExecProcessManager::exec_command(...)`
   - manager 真正怎么把 request 变成 process session
