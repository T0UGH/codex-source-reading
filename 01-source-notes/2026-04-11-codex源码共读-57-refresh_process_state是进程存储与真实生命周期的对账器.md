---
title: refresh_process_state(...) 是进程存储与真实生命周期的对账器
date: 2026-04-11
tags:
  - Codex
  - 源码共读
  - unified-exec
  - process-store
  - lifecycle
source_files:
  - /Users/wangguiping/workspace/codex/codex-rs/core/src/unified_exec/process_manager.rs
status: done
---

# Codex 源码共读 57：refresh_process_state(...) 是进程存储与真实生命周期的对账器

## 这篇看什么

`refresh_process_state(...)` 不是创建进程，也不是输出 watcher，
但它在 unified-exec 里很关键。

因为只要涉及：
- 初次 `exec_command(...)` 返回时，这个 process 现在算活着还是已经退出
- `write_stdin(...)` / poll 之后，这个 process 还在不在 store 里
- 网络审批等 sidecar 该不该清理

最后都要经过它来拍板。

## 先给主结论

如果先留一句话，我会写：

> `refresh_process_state(...)` 的本质不是读取缓存状态，而是 unified-exec process store 与真实进程生命周期之间的对账器：它在持锁查看 store 时，先判断进程是否已退出；若已退出，就当场把 entry 从 store 中摘掉并返回 `Exited { entry, exit_code }`，随后在锁外完成 network approval 清理；若仍存活，则返回 `Alive`；若 store 里根本没有，则返回 `Unknown`。也就是说，它不是被动 getter，而是一次带副作用的状态收口操作。

再压缩一点：

> **它不是查状态，而是边查边收口。**

---

## 第一层：它返回的不是进程状态快照，而是“对账后的 store 状态”

实现里最关键的是：
- 先从 `store.processes.get(&process_id)` 找 entry
- 看 `entry.process.has_exited()`
- 如果退出，就 `store.remove(process_id)` 再返回 `Exited`

这意味着它返回的状态，不是简单观测值，
而是：

> **在完成一次 store reconciliation 之后的结果。**

这就是为什么我说它是对账器，不是 getter。

---

## 第二层：`Exited` 分支把 `entry` 整个带出来，说明“退出”不仅是布尔状态，还要携带清理语义

如果只是要说“死了”，
返回个 `bool` 或 `exit_code` 就够了。

但这里返回的是：
- `ProcessStatus::Exited { exit_code, entry: Box<ProcessEntry> }`

这说明退出之后，上层还需要这份 entry 去做别的事。

源码里立即能看到一个：
- `unregister_network_approval_for_entry(entry).await`

所以 `Exited` 不是纯状态，
而是：

> **退出事实 + 后续清理上下文**

这就是成熟的生命周期设计。

---

## 第三层：它故意把异步清理放到锁外做

这点非常对。

代码先在锁内：
- 查 entry
- 看 exit
- 必要时 remove
- 构造 `status`

锁放掉之后才：
- `if let ProcessStatus::Exited { entry, .. } => unregister_network_approval_for_entry(entry).await`

这说明作者非常清楚：

> **process_store 锁只保护状态对账，不承载异步副作用。**

否则很容易变成长锁、死锁风险、或者让 store 成为瓶颈。

这是典型的好工程判断。

---

## 第四层：`Unknown` 不是异常值，而是有业务语义的第三态

这里不是二态：
- Alive / Exited

还有：
- Unknown

它表示的是：
- store 里压根没有这个 process_id

这在 unified-exec 里非常有意义，
因为它可能代表：
- 从没存在过
- 已经被 earlier refresh/remove 清掉
- 被别的路径 release 掉了

所以 `Unknown` 的作用是：

> **明确区分“已知退出”与“根本无此记录”。**

这个区分对 API 行为很重要。

---

## 第五层：它是 `exec_command(...)` 和 `write_stdin(...)` 的共同收口点

调用位置看得很清楚。

### 在 `exec_command(...)`
它决定：
- 返回里还要不要带 `process_id`
- 进程是不是已经短命退出了
- 如果已经退出，调用方要不要把它当 long-lived session 看

### 在 `write_stdin(...)`
它决定：
- poll 后是 `Alive` 还是 `Exited`
- `TerminalInteraction` 该带哪个 `event_call_id`
- 是否应返回 `UnknownProcessId`

所以它其实是 unified-exec 对“会话是否仍然存在”的共享裁决口。

---

## 第六层：它反映了 unified-exec 的 store 不是 source of truth，而是可回收缓存层

这点很重要。

如果 store 是绝对真相源，
那它只需要读 store 里的状态字段就好了。

但这里不是。

它会直接问：
- `entry.process.has_exited()`
- `entry.process.exit_code()`

然后反过来修正 store。

这说明真正更靠近真相的是：
- `UnifiedExecProcess`

而 `process_store` 更像：
- 对外可寻址的 live-session registry

`refresh_process_state(...)` 就是两者之间的同步点。

---

## 第七层：这里也再次体现 Codex 很喜欢“小对账器”而不是“大总管”

这个函数又是一个典型例子：
- 小
- 边界清楚
- 带一点副作用
- 但决定了上层很多行为正确性

这和 app-server 那些：
- `resolve_thread_status(...)`
- `track_current_turn_event(...)`
- `find_and_remove_turn_summary(...)`

风格非常一致。

我的判断越来越稳：

> Codex 的一致性很多时候来自一批小型 reconciliation functions，而不是一棵巨大的中心状态机。

---

## 一句角色定义

如果按源码共读风格压一句话，我会这么写：

> **`refresh_process_state(...)` 是 unified-exec process store 与真实进程生命周期之间的对账器：它边检查、边摘除已退出进程，并把后续清理所需上下文一并交给调用方。**

---

## 为什么这层设计是对的

### 1. 查询与收口合一
调用方不用重复写清理逻辑。

### 2. 锁内只做同步状态修正，锁外做异步副作用
并发边界清楚。

### 3. `Unknown` 与 `Exited` 分开
API 语义更明确。

### 4. 让 store 保持干净
避免死进程条目长期残留。

---

## 继续细拆的话，下一篇最自然的点

最自然继续拆的是：

1. `handle_user_message(...)`
2. `HeadTailBuffer`
3. `prepare_process_handles(...)`
