---
title: reconstruct_history_from_rollout(...) 是恢复态重建器
date: 2026-04-11
tags:
  - Codex
  - 源码共读
  - rollout
  - resume
  - history
  - state-reconstruction
source_files:
  - /Users/wangguiping/workspace/codex/codex-rs/core/src/codex/rollout_reconstruction.rs
  - /Users/wangguiping/workspace/codex/codex-rs/core/src/codex.rs
  - /Users/wangguiping/workspace/codex/codex-rs/core/src/context_manager/history.rs
  - /Users/wangguiping/workspace/codex/codex-rs/core/src/codex/rollout_reconstruction_tests.rs
status: done
---

# Codex 源码共读 34：reconstruct_history_from_rollout(...) 是恢复态重建器

## 这篇看什么

前面的 rollout/state_db 架构稿已经立住了：
- rollout 是正文真相源
- state_db 是 metadata/index sidecar
- 恢复不能只靠 SQLite

但这些都还是系统级判断。

真正更细的问题是：

> **从 rollout items 到内存里的 history，到底是怎么一步步重建出来的？**

更具体一点：
- 为什么它要先 reverse scan，再 forward replay
- `replacement_history` 到底在恢复里扮演什么角色
- rollback 为什么不能只在最后粗暴删 turn
- `previous_turn_settings` 和 `reference_context_item` 为什么要和 history 一起重建

这篇只讲一个函数：

- `Session::reconstruct_history_from_rollout(...)`

## 先给主结论

如果先留一句话，我会写：

> `reconstruct_history_from_rollout(...)` 不是“把 rollout 转成 history”这么简单，它实际上是 Codex 恢复态里的统一重建器：先从新到旧扫描，找出**最新仍然有效的 history checkpoint** 和 **最新仍然有效的 turn metadata/baseline**，同时把 rollback 的影响在 metadata 层也消化掉；然后再只对幸存后缀做一次正向 replay，生成真正的 transcript history，最后把这些结果一起装配成 resume/fork 所需的内存状态。

再压缩一点：

> **它不是单纯回放，而是在做“checkpoint 选择 + metadata 对齐 + 精确 replay”的恢复重建。**

---

## 第一层：它返回的不是一份 history，而是一组恢复态

这个函数返回的是：
- `history`
- `previous_turn_settings`
- `reference_context_item`

这点特别关键。

说明在 Codex 看来，恢复一个 session 不只是：
- 把消息列表拼出来

而是要同时恢复：
- 上一轮的 model/realtime 设置
- 当前 context baseline
- transcript 本身

所以它更像：

> **resume/fork hydration function**

而不是普通的“日志回放函数”。

---

## 第二层：它故意分成两段，而不是一次从头 replay

我觉得这函数最重要的结构是：

### 段 1：reverse scan
目标不是直接重建 transcript，
而是找到：
- 最新 surviving `replacement_history`
- 最新 surviving `previous_turn_settings`
- 最新 surviving `reference_context_item`
- rollback 之后哪些 segment 还算活着

### 段 2：forward replay
从选出来的 checkpoint/base 开始，
只对幸存后缀做一次正向 replay，
把 `ContextManager` history 精确 materialize 出来。

这个设计特别像成熟系统里常见的：

> **先找最小必要起点，再精确回放。**

而不是从全量历史一把梭。

---

## 第三层：`replacement_history` 不是普通字段，而是 checkpoint

这点必须单独讲。

遇到 `Compacted { replacement_history: Some(...) }` 时，
函数不会立刻说“好了，历史就是它”。

而是先把它挂到当前 reverse segment 上，
等 segment finalize 时再决定它是否存活。

### 为什么不能立刻认定它有效
因为 rollback 可能把这个 segment 整个吃掉。

只有 segment survive 之后，这个 `replacement_history` 才有资格成为：
- 全局 `base_replacement_history`

这说明作者非常清楚：

> checkpoint 不是“看到了就有效”，而是“看到了且还活着才有效”。

这个细节非常工程化。

---

## 第四层：reverse scan 真正的单位不是 item，而是 turn segment

函数里有个特别值的结构：
- `ActiveReplaySegment`

它会累计一个 segment 里的：
- turn_id
- counts_as_user_turn
- previous_turn_settings
- reference_context_item
- base_replacement_history

然后直到看到匹配的 `TurnStarted` 才 finalize。

这说明 reverse scan 不是：
- 一条条 item 独立判断

而是：
- **按 turn segment 聚合后再判断这个 segment 是否还算活着、贡献什么 metadata**

这一步是理解整个函数的关键。

因为 rollback 的语义其实就是：
- 删除“最近 N 个用户 turn segment”

而不是删除某几条 item。

---

## 第五层：rollback 被分成两次处理，是故意的

### 第一次：reverse metadata 层
通过 `pending_rollback_turns`，
在 finalize segment 时，如果这个 segment 是 user turn，就把它跳过。

这样可以保证：
- 被 rollback 掉的 turn
- 不会再贡献 `previous_turn_settings`
- 不会再贡献 `reference_context_item`
- 它携带的 checkpoint 也不会被采用

### 第二次：forward transcript 层
在 forward replay 时，再对真正的 transcript history 调：
- `drop_last_n_user_turns(...)`

这两层都要做，原因很明确：

> **不仅 transcript 要和 rollback 对齐，metadata 也必须和 rollback 对齐。**

如果只在最后删 transcript，
那 `previous_turn_settings` 很可能还指向一个已经被删掉的 turn。

这就是为什么这个函数比“简单 replay”高级一层。

---

## 第六层：`reference_context_item` 的三态设计很值

内部不是直接 `Option<TurnContextItem>`，
而是：
- `NeverSet`
- `Cleared`
- `Latest(...)`

### 为什么不是直接 `Some/None`
因为：
- `NeverSet` 表示“我还没看到任何证据”
- `Cleared` 表示“之前可能有，但后面的 compaction 已经明确把它作废了”

这两者最后都可能映射成 `None`，
但在 reverse scan 过程中语义完全不同。

这个设计说明作者不是在写方便代码，
而是在认真区分：

> “没有信息” 和 “明确被清掉了”

这在恢复逻辑里非常重要。

---

## 第七层：legacy compaction 为什么会让 baseline 清空
如果碰到没有 `replacement_history` 的 legacy compaction，
函数虽然还能 rebuild history，
但最后会强制：
- `reference_context_item = None`

这其实是一个非常保守但正确的判断。

意思是：

> transcript 我还能尽量恢复，
> 但 baseline 语义我已经不能再自信地保留了，
> 所以下一轮宁可 full reinject，也不要假装 diff baseline 还可信。

这个选择很成熟。

---

## 第八层：forward replay 没有自己拼 vec，而是回到 `ContextManager`
forward 阶段不是手写一堆 list append/remove，
而是走：
- `history.record_items(...)`
- `history.replace(...)`
- `history.drop_last_n_user_turns(...)`

这说明函数不想重新发明一套“恢复时专用的 history 语义”，
而是尽量让恢复复用平时真实运行时的 transcript 规则。

这个决定很对。

否则会出现：
- live session 的 history 语义一套
- resumed session 的 history 语义另一套

这是非常难 debug 的坑。

---

## 第九层：这个函数最核心的角色，不是“恢复 transcript”，而是“对齐 transcript 和 metadata”

如果只说 transcript replay，这函数的复杂度显得很夸张。

但如果把它理解成：

> **确保 transcript、previous_turn_settings、reference_context_item 三者来自同一份 surviving 历史现实**

那复杂度就完全合理了。

这也是我觉得这函数最值得学习的地方。

它不是为了代码炫技才复杂，
而是因为恢复态里最危险的问题从来不是“少一条消息”，
而是：
- transcript 是新的
- baseline 是旧的
- settings 又来自被 rollback 的 turn

这种三者错位。

这个函数就是在防这个坑。

---

## 一句角色定义

如果要用一句最短的话来讲，我会写：

> **`reconstruct_history_from_rollout(...)` 是 Codex 的恢复态重建器：它先从 rollout 里判断“哪些历史和 metadata 还有效”，再用最小必要的 forward replay 把 transcript 和 baseline 一起对齐回内存。**

---

## 继续细拆的话，下一篇最自然的点

如果继续往下拆，最自然的是两刀：

1. `finalize_active_segment(...)`
   - rollback / checkpoint / metadata 采用顺序为什么这样排
2. `apply_rollout_reconstruction(...)`
   - reconstructed history 怎样真正灌回 `Session`
