---
title: spawn_exit_watcher(...) 与 start_streaming_output(...) 是终态发射链
date: 2026-04-11
tags:
  - Codex
  - 源码共读
  - unified-exec
  - watcher
  - output
  - lifecycle
source_files:
  - /Users/wangguiping/workspace/codex/codex-rs/core/src/unified_exec/async_watcher.rs
  - /Users/wangguiping/workspace/codex/codex-rs/core/src/unified_exec/process_manager.rs
status: done
---

# Codex 源码共读 49：spawn_exit_watcher(...) 与 start_streaming_output(...) 是终态发射链

## 这篇看什么

前面 `from_spawned(...)` 和 `store_process(...)` 已经把 unified-exec 的前半段讲清了：
- raw spawn 如何变成 session
- session 如何进入 manager store

但更细的问题是：

> **一个活着的 unified-exec session，最后是怎么优雅地结束、排干输出、并发出最终 end event 的？**

这篇把两个函数放在一起讲：

- `start_streaming_output(...)`
- `spawn_exit_watcher(...)`

因为它们其实是一条终态发射链的前后半段。

## 先给主结论

如果先留一句话，我会写：

> `start_streaming_output(...)` 和 `spawn_exit_watcher(...)` 不是两个独立 watcher，而是一条前后接力的终态发射链：前者负责持续订阅 process output、做 UTF-8 安全切块、写 transcript、发增量 delta，并在 exit/cancellation 后等待一个 trailing-output grace 再宣告 `output_drained`；后者则在 `exit_token.cancelled()` 和 `output_drained` 两个门都过了之后，才最终发出 `ExecCommandEnd`。也就是说，Codex 的 unified-exec 终态不是“进程一退出就结束”，而是“退出 + 输出排干”之后才算真的结束。

再压缩一点：

> **前者排输出，后者封终态。**

---

## 第一层：`start_streaming_output(...)` 真正负责的是 transcript 和 delta，不是终态

它的工作可以概括成：
- 订阅 process output broadcast
- merge chunk 到 transcript buffer
- 做 UTF-8 safe split
- 发 `ExecCommandOutputDelta`
- 在 exit 后给 trailing output 一个 grace 窗口
- 最后通知 `output_drained`

也就是说，它并不发最终 end event。

这说明它的定位很明确：

> **它是 live-output sidecar，不是终态判决器。**

---

## 第二层：为什么它要在 exit 后还等 100ms

这一步特别值。

`start_streaming_output(...)` 一旦看到：
- cancellation token 触发

它不会立刻退出，
而是会：
- 启动一个 100ms 的 `TRAILING_OUTPUT_GRACE`

原因非常清楚：
- 进程退出信号到了
- 但输出 channel/buffer 里可能还有尾巴没完全送完

所以这一步本质是在说：

> **退出 != 输出已经排干。**

这是很多终端/进程系统很容易做错的一点，
Codex 这里是明显认真处理了这个尾巴窗口。

---

## 第三层：`output_drained` 是这条链里最关键的接力信号

前半段 watcher 最后会在两个时机之一发：
- grace 结束
- output channel 关闭

然后 `notify output_drained`

这就是交接棒。

后半段 `spawn_exit_watcher(...)` 做的第一件事，不是直接发 end，
而是：
1. 等 `exit_token.cancelled()`
2. 再等 `output_drained.notified()`

所以 end event 的真正触发条件不是单一的进程退出，
而是：

> **进程退出 + 输出排干**

这就是一条非常清晰的双门模型。

---

## 第四层：这意味着 unified-exec 的 end event 是“语义完成”，不是“OS 进程退出”

这点很值得单独说。

如果系统简单，它可以：
- child 退出
- 马上 emit end

但 Codex 不是这么做的。

现在的语义其实是：
- OS process 退出只是第一门
- transcript/output 已稳定才是第二门
- 两门都过了，才 emit `ExecCommandEnd`

所以这里的 end event 表示的是：

> **这次 command 对用户和上层 runtime 来说，已经完整收口了。**

这是比“进程结束”更高一级的语义。

---

## 第五层：`start_streaming_output(...)` 和 process 内部 output buffer 不是同一个东西

这个点很容易忽略。

统一进程对象里本来就有自己的 output buffer，
而 `start_streaming_output(...)` 还会额外维护一份 transcript `HeadTailBuffer`。

也就是说，这里其实有两条输出路径：

1. process 内部 retained output buffer
2. watcher 侧 transcript buffer

这说明 unified-exec 的设计不是单一缓存桶，
而是：
- 一条偏 session/process 语义
- 一条偏事件/最终 end payload 语义

这也解释了为什么 end watcher 会依赖 transcript，而不是直接去 process 内部拿所有东西。

---

## 第六层：broadcast lag 设计说明这条链优先保证“系统活着”，其次才是“流式完美无损”

在 `start_streaming_output(...)` 里，如果 broadcast receiver lag 了：
- 会直接 continue

这意味着：
- live delta 不是绝对无损保证
- transcript 这一路也可能因此不是绝对全量

所以这条链的设计取舍更像：

> **优先有界、可持续地流式更新，而不是承诺每个 chunk 在 watcher 侧都绝对不丢。**

这是一个很现实的工程取舍。

---

## 第七层：为什么 `spawn_exit_watcher(...)` 必须独立出来，而不是塞进输出 watcher 里

因为两者职责不同：

### `start_streaming_output(...)`
- 持续读 output
- 处理 chunk
- 管 transcript 与 delta

### `spawn_exit_watcher(...)`
- 等待终态条件
- 计算 duration
- 决定 success/failure payload
- emit 最终 end event

如果把两者揉在一起，代码会非常混乱。

现在拆开之后，模型很清楚：

> **一个 watcher 负责“输出什么时候算写完”，另一个 watcher 负责“什么时候可以安全宣布结束”。**

这就是成熟的生命周期拆分。

---

## 第八层：失败终态和成功终态共享 end 通道，但 payload 组装不同

`spawn_exit_watcher(...)` 最后会分两条：

### 正常成功
- `emit_exec_end_for_unified_exec(...)`
- 最终还是发 `ExecCommandEnd`

### 失败
- `emit_failed_exec_end_for_unified_exec(...)`
- 也还是 `ExecCommandEnd`
- 但会把 failure message 带进去，exit_code 也不同

这说明 unified-exec 对外的终态协议是：

> **终态事件统一，成功/失败通过 payload 区分。**

这和 app-server turn completed 那种“统一 completed 事件，靠 status 区分”非常像。

说明 Codex 很喜欢这种：
- 终态通道统一
- 细语义靠 payload/status 区分

---

## 第九层：这一对函数本质上是“output lifecycle”和“process lifecycle”的对接点

`start_streaming_output(...)` 更偏：
- output lifecycle

`spawn_exit_watcher(...)` 更偏：
- process lifecycle

而它们通过：
- cancellation token
- output_drained

接起来。

这说明 unified-exec 的终态并不是只围绕 process，而是围绕：

> **process lifecycle × output lifecycle 的汇合点。**

这也是为什么这两个函数必须放在一起看。

---

## 一句角色定义

如果按源码共读风格压一句话，我会这么写：

> **`start_streaming_output(...)` 和 `spawn_exit_watcher(...)` 共同组成了 unified-exec 的终态发射链：前者保证输出被尽可能排干并落进 transcript，后者在确认“退出 + 排干”之后，再把最终 end event 发出去。**

---

## 为什么这层设计是对的

### 1. 不把 end event 绑死在 OS process exit 上
这样对用户更真实。

### 2. 把 trailing output grace 明确建模
避免最后一小段输出丢掉。

### 3. 输出 watcher 和终态 watcher 分工明确
职责边界更清晰。

### 4. 统一成功/失败终态通道
协议对外更稳定。

---

## 继续细拆的话，下一篇最自然的点

最自然继续拆的是：

1. `process_chunk(...)`
2. `emit_exec_end_for_unified_exec(...)`
3. `emit_failed_exec_end_for_unified_exec(...)`

这三篇会把 unified-exec 的输出与终态发射链彻底讲透。
