---
title: find_and_remove_turn_summary(...) 是 turn 局部状态的 drain 边界
date: 2026-04-11
tags:
  - Codex
  - 源码共读
  - app-server
  - turn-summary
  - finalization
source_files:
  - /Users/wangguiping/workspace/codex/codex-rs/app-server/src/bespoke_event_handling.rs
  - /Users/wangguiping/workspace/codex/codex-rs/app-server/src/thread_state.rs
status: done
---

# Codex 源码共读 45：find_and_remove_turn_summary(...) 是 turn 局部状态的 drain 边界

## 这篇看什么

前面 `handle_turn_complete(...)` 已经知道：
- 它会依赖 `turn_summary.last_error`
- 它会读取 `started_at`
- turn 终态不是 terminal event 单点决定

但真正再往下一层的问题是：

> **turn summary 到底在哪一刻从“正在积累的局部状态”切换成“已消费的历史”？**

这篇只讲：
- `find_and_remove_turn_summary(...)`

## 先给主结论

如果先留一句话，我会写：

> `find_and_remove_turn_summary(...)` 的真正价值不在“查找”，而在“移除”：它通过 `std::mem::take(&mut state.turn_summary)` 把当前 turn 正在累积的局部状态一次性抽走，并立即把 thread state 里的 summary slot 复位成默认值。这样一来，`handle_turn_complete(...)`、`handle_turn_interrupted(...)` 等后续逻辑拿到的是本轮 turn 的完整快照，而 thread state 同时也完成了为下一轮 turn 清场的切换。这不是普通 getter，而是 turn finalization 里的 drain 边界。

再压缩一点：

> **它不是取 summary，而是把本轮 summary 正式结账。**

---

## 第一层：它真正做的不是 find，而是 take

函数名里有 `find`，但实际行为几乎全在：
- `std::mem::take(&mut state.turn_summary)`

这一步的语义和普通读取完全不同。

普通读取只是：
- 看一眼当前状态

这里的 `take` 是：
- 取出当前完整 summary
- 并把内部 slot 重置成 `TurnSummary::default()`

所以它的真实意义不是“找到了哪个 summary”，
而是：

> **把这个 turn 的局部累积状态从 live state 中摘下来。**

这就是 drain boundary。

---

## 第二层：为什么这一步必须是 destructive read

`TurnSummary` 里存的不是纯展示数据，而是本轮运行中的临时状态：
- `started_at`
- `last_error`
- `file_change_started`
- `command_execution_started`

如果这里只是读，不清掉，
下一轮 turn 就很容易继承上一轮残留：
- 老的 `last_error`
- 老的 started item ids
- 老的 `started_at`

所以这里用 `mem::take(...)` 的真正理由是：

> **turn-local summary 是有生命周期边界的，而这个函数就是那个边界。**

---

## 第三层：它其实是 turn finalization 闭环里的状态切换器

放在链路里看，它的位置特别清楚：

### 写入侧
- `track_current_turn_event(...)` 写 `started_at`
- `handle_error(...)` 写 `last_error`
- patch/exec item 分支写 started sets

### 读取/结算侧
- `handle_turn_complete(...)`
- `handle_turn_interrupted(...)`

而 `find_and_remove_turn_summary(...)` 正好卡在：

> **从“还在累积”切到“拿去结算”的那个瞬间。**

它不是业务逻辑本身，但没有它，前后两段就接不上。

---

## 第四层：它一次性清掉的不只是 error/timestamp，还包括 item-state 去重集合

这一点很容易被忽略。

`TurnSummary` 里的两个 set：
- `file_change_started`
- `command_execution_started`

平时在 turn 内负责：
- suppress duplicate `ItemStarted`
- 帮助 output delta 做 item 类型判断

而 `mem::take(...)` 一下子会把它们也清掉。

这说明 turn finalization 在作者心里不只是：
- “这轮完成了，记录个时间”

而是：
- “这轮相关的所有局部 bookkeeping 到这里都该结账归零了。”

这非常完整。

---

## 第五层：它的 `_conversation_id` 参数没用，反而暴露了一个真实边界

这个函数接了 `conversation_id`，但没用。

这看似奇怪，其实反而说明了一个事实：

> **真正决定 summary 归属的不是参数，而是调用方传进来的那个 `Arc<Mutex<ThreadState>>`。**

也就是说，thread 级隔离已经在更外层保证好了。

这个函数本身不关心“我要去哪个 map 里找”，
它只管：
- 给我的这个 `thread_state`，把 summary take 出来

所以从分层角度看，它是很纯的局部 state helper。

---

## 第六层：这函数让 `handle_turn_complete(...)` 和 `handle_turn_interrupted(...)` 都能共享同一个 drain 语义

无论最终 turn 是：
- `Completed`
- `Failed`
- `Interrupted`

这两条路径都会先调：
- `find_and_remove_turn_summary(...)`

这说明作者不想让“清状态”散落到多个终态处理器里，
而是明确抽成一个共享 drain 入口。

这很合理。

不然很容易出现：
- complete 路径清了
- interrupted 路径忘了清
- 或清理顺序不一致

现在这个 helper 把这件事统一了。

---

## 第七层：这个函数最像“状态机 commit 点”，而不是普通 getter

如果用更状态机的语言讲，
它像是在做：

> **把当前 turn 的 working set commit 成 finalization input，并清空 working slot。**

它不是数据库意义上的 commit，
但从语义上非常像：
- 这轮暂存状态到这里算封板
- 之后就该进入 terminal 判决与发射阶段

所以我觉得把它叫 getter 会严重低估它。

---

## 一句角色定义

如果按源码共读风格压一句话，我会这么写：

> **`find_and_remove_turn_summary(...)` 不是读取 turn summary 的函数，而是 app-server turn finalization 里把本轮局部状态整体 drain 出来、并为下一轮清空 slot 的生命周期边界函数。**

---

## 为什么这层设计是对的

### 1. 把 turn summary 生命周期边界显式抽出来
这会让 complete/interrupted 路径更一致。

### 2. 用 `mem::take(...)` 一次性清空所有局部 bookkeeping
防止 turn 间串味。

### 3. 让 summary 读和 summary 清场在同一步发生
减少状态机不一致风险。

---

## 继续细拆的话，下一篇最自然的点

最自然继续拆的是：

1. `track_current_turn_event(...)`
2. `complete_command_execution_item(...)`
3. `complete_file_change_item(...)`

这几篇会把 `TurnSummary` 的写入/消费链彻底闭环。
