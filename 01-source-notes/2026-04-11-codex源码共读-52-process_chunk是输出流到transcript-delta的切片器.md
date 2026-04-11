---
title: process_chunk(...) 是输出流到 transcript/delta 的切片器
date: 2026-04-11
tags:
  - Codex
  - 源码共读
  - unified-exec
  - output
  - utf8
  - transcript
source_files:
  - /Users/wangguiping/workspace/codex/codex-rs/core/src/unified_exec/async_watcher.rs
  - /Users/wangguiping/workspace/codex/codex-rs/core/src/unified_exec/async_watcher_tests.rs
status: done
---

# Codex 源码共读 52：process_chunk(...) 是输出流到 transcript/delta 的切片器

## 这篇看什么

前面已经知道：
- `start_streaming_output(...)` 负责持续读 output channel
- `spawn_exit_watcher(...)` 负责最终发 end event

那两者中间真正把“收到的一坨 bytes”变成：
- transcript 累积内容
- `ExecCommandOutputDelta` 增量事件

的，就是 `process_chunk(...)`。

## 先给主结论

如果先留一句话，我会写：

> `process_chunk(...)` 的本质不是 append buffer，而是 unified-exec 流式输出的切片器：它把原始 byte chunk 先并入 `pending`，然后按 UTF-8 有效前缀逐段切出可消费片段，每一段都先落进 transcript，再在事件预算未超限时发 `ExecCommandOutputDelta`。所以它维护的是“完整 transcript 持续累积”与“流式事件有界发射”这两个目标的平衡，而不是简单的字节转发。

再压缩一点：

> **它负责把原始字节流切成可展示文本片段，并且区分 transcript 完整性与 delta 节流。**

---

## 第一层：它先写 transcript，再考虑要不要发 delta

代码顺序很关键：

1. `pending.extend_from_slice(&chunk)`
2. `while let Some(prefix) = split_valid_utf8_prefix(pending)`
3. 对每个 `prefix`：
   - 先 `transcript.push_chunk(...)`
   - 再看 `emitted_deltas` 是否超限
   - 没超限才发 `ExecCommandOutputDelta`

这说明它的优先级很明确：

> **完整保留 transcript > 实时 delta 是否还能继续发。**

也就是说，live event stream 可以被截流，
但 transcript 不能因为事件预算耗尽就停写。

这是成熟的分层设计。

---

## 第二层：`pending` 的存在说明 chunk 边界不可信，必须自己重切

外面传进来的 `chunk: Vec<u8>` 只是 transport / output receiver 的分块结果，
并不保证：
- 正好落在 UTF-8 字符边界上
- 适合直接发给上层

所以 `process_chunk(...)` 不会直接发 chunk，
而是：
- 先拼到 `pending`
- 再用 `split_valid_utf8_prefix(...)` 反复找“当前能安全吐出的前缀”

这说明作者明确承认：

> **底层 IO chunk 边界不是产品语义边界。**

这个判断是对的。

---

## 第三层：它追求的是 UTF-8 安全切块，不是原样复刻底层分块

测试已经把这个意图写死了：
- ASCII 会按 `max_bytes` 切
- 多字节字符不会被从中间截断
- 即便遇到非法 UTF-8，也会至少吐 1 byte 保证进展

所以这套切片规则的目标不是：
- 完美复刻输入 chunk

而是：
- **尽量以 UTF-8 合法文本片段为单位向上游暴露内容**
- **同时保证不会因为坏字节卡死整个流**

这很像一个有容错的 decoder stage。

---

## 第四层：它把“有界增量事件”和“无界 transcript 累积”显式分离了

这里最值的一点是：
- `MAX_EXEC_OUTPUT_DELTAS_PER_CALL` 只约束 delta 事件数
- 不约束 transcript 累积

所以一旦 delta 发射达到预算：
- 它会 `continue`
- 但前面已经写入 transcript 了

这其实是在做一个很清楚的工程取舍：

> **实时性可以有上限，但最终完整输出不能因此丢。**

这一点和普通“边读边发”差别很大。

---

## 第五层：它是 unified-exec 中最典型的“局部小函数承担协议质量”例子

`process_chunk(...)` 很短，
但实际承担了几件很重的事：
- UTF-8 边界安全
- transcript 累积
- live event 发射
- 发射预算控制
- invalid bytes 不阻塞

这说明 unified-exec 的质量不是靠一个大函数兜住，
而是靠这种小函数把关键边界处理扎实。

这和 app-server 那边很多“小修正器”的风格很一致。

---

## 第六层：它把 stdout 语义硬编码在 delta 事件里，说明 unified-exec 当前没有精细区分流源

生成 `ExecCommandOutputDeltaEvent` 时，`stream` 固定写的是：
- `ExecOutputStream::Stdout`

这意味着当前这条 watcher 链路里：
- 输出增量事件更偏“统一输出流”语义
- 不是精细保留 stdout/stderr 双流细分

而最终 end payload 里也能看到：
- success 路径基本把 aggregated output 都放进 stdout / aggregated_output

这说明 unified-exec 目前更重视：
- 对 agent/runtime 的统一消费体验

而不是终端级别的严格双流还原。

这是个明确 trade-off。

---

## 第七层：它其实是 output lifecycle 里最关键的可逆点

所谓可逆点，是指：
- 从这里往前还是原始 bytes / receiver chunk
- 从这里往后就已经是 transcript / protocol event 了

一旦经过 `process_chunk(...)`，输出就被转译成：
- 可聚合 transcript
- 可发送协议增量

所以它其实是 unified-exec 输出路径上的一个小“codec boundary”。

---

## 一句角色定义

如果按源码共读风格压一句话，我会这么写：

> **`process_chunk(...)` 是 unified-exec 输出路径上的切片器：它把不可信的原始字节 chunk 重整成 UTF-8 安全的 transcript 片段，并在预算允许时转发为增量 delta 事件。**

---

## 为什么这层设计是对的

### 1. transcript 和 delta 明确分层
完整性与实时性可以分别优化。

### 2. 不信任底层 chunk 边界
对真实 IO 更稳。

### 3. UTF-8 安全 + 非法字节可进展
不会卡死输出流。

### 4. 小函数承担关键边界质量
可读且容易测试。

---

## 继续细拆的话，下一篇最自然的点

最自然继续拆的是：

1. `split_valid_utf8_prefix(...)`
2. `resolve_aggregated_output(...)`
3. `emit_exec_end_for_unified_exec(...)`
