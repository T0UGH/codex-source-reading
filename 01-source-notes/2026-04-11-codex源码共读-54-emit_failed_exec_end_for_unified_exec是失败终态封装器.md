---
title: emit_failed_exec_end_for_unified_exec(...) 是失败终态封装器
date: 2026-04-11
tags:
  - Codex
  - 源码共读
  - unified-exec
  - exec-end
  - failure
source_files:
  - /Users/wangguiping/workspace/codex/codex-rs/core/src/unified_exec/async_watcher.rs
  - /Users/wangguiping/workspace/codex/codex-rs/core/src/unified_exec/process_manager.rs
status: done
---

# Codex 源码共读 54：emit_failed_exec_end_for_unified_exec(...) 是失败终态封装器

## 这篇看什么

前一篇已经拆了 success 分支：
- `emit_exec_end_for_unified_exec(...)`

那对应的失败分支，就是：
- `emit_failed_exec_end_for_unified_exec(...)`

它既会被后台 `spawn_exit_watcher(...)` 调用，
也会被 `exec_command(...)` 在“进程没活成长期 session 就直接失败”的路径调用。

## 先给主结论

如果先留一句话，我会写：

> `emit_failed_exec_end_for_unified_exec(...)` 的本质不是“报错时发个 end”，而是 unified-exec 失败终态的标准封装器：它先把 transcript 中已经成功累积的 stdout 与 failure message 合并成最终 aggregated output，再统一装配成 `ToolEventStage::Failure(ToolEventFailure::Output(output))`，最终仍通过同一个 `ToolEmitter::unified_exec(...)` 发成 `ExecCommandEnd`。也就是说，Codex 不把失败当成协议旁路，而是把它纳入与成功同构的终态发射模型里。

再压缩一点：

> **失败不是旁路异常，而是同构终态。**

---

## 第一层：它和成功封装器是平行结构，不是附属补丁

成功函数和失败函数骨架非常像：
- 都拿 transcript
- 都构造 `ExecToolCallOutput`
- 都创建 `ToolEventCtx`
- 都走 `ToolEmitter::unified_exec(...)`
- 都最终发 `ExecCommandEnd`

差别只在：
- success 用 `ToolEventStage::Success`
- failure 用 `ToolEventStage::Failure(ToolEventFailure::Output(...))`

这说明设计意图很明确：

> **成功/失败共享统一终态通道，差异留在 stage/payload 层。**

这比“失败另起一类 event”成熟得多。

---

## 第二层：它先保留已有 stdout，再拼上 failure message

实现里先做：
- `stdout = resolve_aggregated_output(&transcript, String::new())`

然后：
- 如果 stdout 为空：`aggregated_output = message`
- 否则：`aggregated_output = stdout + "\n" + message`

这一步特别值。

因为它明确承认：
- 一个失败的 exec，不代表前面没有产生有效输出

所以失败终态不是只剩错误，
而是：

> **已有输出 + 最终失败说明 的组合。**

这对调试和 agent 判断都更真实。

---

## 第三层：失败路径里 stdout / stderr 的分工比成功路径清楚得多

成功路径基本把聚合结果都塞进 stdout/aggregated_output，stderr 为空。

而失败路径里：
- `stdout` = transcript 中已有输出
- `stderr` = failure message
- `aggregated_output` = 二者拼接结果

这说明 unified-exec 在失败场景下，
其实愿意给出更清楚的语义分层：

> **正常输出仍是 stdout，失败说明单独进入 stderr，同时 aggregated_output 提供整体消费视图。**

这是一个很好的失败设计。

---

## 第四层：固定 `exit_code = -1` 说明这里表达的是“runtime-level failure”，不是底层进程真实退出码

这里没有去拿真实退出码，
而是直接：
- `exit_code: -1`

这说明它表达的不是：
- OS process 以几号状态码退出

而是：
- **这次 unified-exec 从工具/runtime 视角失败了**

所以这个函数处理的是更高层的失败语义。

这是合理的，
因为有些失败本来就不对应一个可依赖的真实子进程退出码。

---

## 第五层：它把 transcript 视为 failure message 之前的“已确认事实”

这里值得记一个顺序感：
- transcript 来自 watcher 持续累积
- failure message 来自 process failure path

最终拼接时，stdout 在前，message 在后。

也就是说作者默认叙事顺序是：
1. 这是执行过程中已经观察到的输出
2. 然后这是最终失败说明

这就是对人类调试者最自然的阅读顺序。

---

## 第六层：调用位置说明它统一了“早失败”和“晚失败”两类路径

### 晚失败
后台 watcher 等到退出 + 输出排干后，发现 `failure_message()`，再调它。

### 早失败
`exec_command(...)` 里如果进程没活成长期 session，就直接调它收口。

所以它的真正价值仍然不是 helper 复用，
而是：

> **无论失败发生在会话建立前还是会话生命周期尾部，对外都长成统一失败终态。**

---

## 一句角色定义

如果按源码共读风格压一句话，我会这么写：

> **`emit_failed_exec_end_for_unified_exec(...)` 是 unified-exec 失败终态的标准封装器：它保留已观察到的 stdout，再把失败说明并入 stderr/aggregated_output，通过统一的 `ExecCommandEnd` 通道把失败语义发出去。**

---

## 为什么这层设计是对的

### 1. 失败不走旁路协议
客户端更稳定。

### 2. 已有输出不因失败而丢失
更利于调试。

### 3. 早失败/晚失败共享同一封装器
语义一致。

### 4. stderr 与 aggregated_output 分工更清楚
agent 和人都更好消费。

---

## 继续细拆的话，下一篇最自然的点

最自然继续拆的是：

1. `resolve_aggregated_output(...)`
2. `split_valid_utf8_prefix(...)`
3. `refresh_process_state(...)`
