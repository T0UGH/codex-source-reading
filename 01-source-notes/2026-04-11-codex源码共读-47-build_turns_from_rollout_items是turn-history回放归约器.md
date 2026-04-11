---
title: build_turns_from_rollout_items(...) 是 turn-history 回放归约器
date: 2026-04-11
tags:
  - Codex
  - 源码共读
  - thread-history
  - rollout
  - replay
source_files:
  - /Users/wangguiping/workspace/codex/codex-rs/app-server-protocol/src/protocol/thread_history.rs
  - /Users/wangguiping/workspace/codex/codex-rs/app-server/src/thread_state.rs
status: done
---

# Codex 源码共读 47：build_turns_from_rollout_items(...) 是 turn-history 回放归约器

## 这篇看什么

前面 `populate_thread_turns(...)` 已经知道：
- 它自己不解释每种 `RolloutItem`
- 真正把 rollout items 变成 turns 的核心在：
- `build_turns_from_rollout_items(...)`

所以这篇就盯这一个点：

> **一串 persisted rollout items，到底是怎样被归约成 v2 `Turn` 列表的？**

## 先给主结论

如果先留一句话，我会写：

> `build_turns_from_rollout_items(...)` 的本质不是简单遍历+拼数组，而是一个状态化的回放归约入口：它真正依赖的是 `ThreadHistoryBuilder` 这套单遍 reducer，把 persisted rollout stream 中混杂的 `EventMsg`、`ResponseItem`、`Compacted` 等项，逐步折叠成可展示的 turn history；显式 turn 边界优先，缺失时再用 user-message 等启发式来补 turn 划分，必要时还会把 late completion 回补到已经 finalized 的旧 turn 里。

再压缩一点：

> **它不是 parser，而是 replay reducer。**

---

## 第一层：这个函数本身很薄，真正的价值在它背后的 builder

`build_turns_from_rollout_items(...)` 本身只做三件事：
1. new 一个 `ThreadHistoryBuilder`
2. 逐项 `handle_rollout_item(...)`
3. `finish()`

这说明它本体不是逻辑重心，
而是把“turn history 回放归约”这个能力作为一个公共入口暴露出来。

所以它真正的意义是：

> **让 `ThreadHistoryBuilder` 成为 live state 和 persisted replay 共用的 reducer 核心。**

这点非常重要。

因为前面已经看到：
- `ThreadState.current_turn_history` 也是它在跑
- persisted rollout replay 还是它在跑

这意味着 Codex 很刻意地在避免：
- live 一套 turn 语义
- replay 一套 turn 语义

---

## 第二层：它的真正执行单位不是 item，而是“builder state + item”

如果只看函数名，很容易以为它在做：
- `Vec<RolloutItem> -> Vec<Turn>`

但真正发生的是：

> **每个 item 都是在推进一个显式 builder state。**

这个 builder 至少有：
- `turns`
- `current_turn`
- `next_item_index`
- rollout index 跟踪

所以这是非常标准的：
- 单遍状态化归约器

而不是纯函数式 map/filter。

---

## 第三层：显式 turn 边界优先，缺失时才靠启发式补 turn

这是这套 reducer 最关键的设计之一。

### 显式 turn 边界
如果 rollout 里有：
- `TurnStarted`
- `TurnComplete`
- `TurnAborted`

那 builder 就优先按这些边界来维护 turn。

### 启发式补 turn
如果缺失显式边界，它才会：
- 在合适位置用 `UserMessage` 等信号触发新的 implicit turn

这说明作者很清楚两类输入来源：
- 新一点的、更结构化的 rollout
- 老一点/不完整的兼容型 rollout

所以 `build_turns_from_rollout_items(...)` 的真正能力不是“正常情况跑通”，
而是：

> **在结构化边界优先的前提下，对旧格式做尽量稳的兼容回放。**

---

## 第四层：late completion 能回补到旧 turn，说明它不是只看当前 turn

这也是很值的一点。

如果某个 turn-scoped item（例如 exec completion）来晚了，
builder 不会只往当前 active turn 塞，
而是会尝试：
- 先匹配 active current turn
- 再匹配已经 finalized 的旧 turn
- 实在找不到才 drop

这说明它的设计不是：
- “只管当前 turn”

而是：
- **把 turn id 当成稳定路由键，允许历史补丁回写。**

这就是为什么它更像 reducer，而不是一次性 parser。

---

## 第五层：compaction-only turn 能保留，说明它不仅在恢复“对话”，也在恢复“语义事件”

builder 对空 turn 不是一刀切丢掉。

如果 turn 虽然没有普通 items，
但和 compaction 语义有关，
它仍然会保留。

这说明它恢复的不只是：
- 聊天内容

还包括：
- 对客户端/系统来说有意义的 turn 级语义节点

这也是“history”比单纯 transcript 更复杂的原因。

---

## 第六层：这里有个很值得记的点——注释和实现可能有轻微漂移

你子 agent 也提到了一个很关键的问题：
- 顶部注释提到 `TurnContext.turn_id`
- 但当前 `RolloutItem::TurnContext(_)` 分支在这里基本是忽略的

这说明：
- 要么注释没跟着代码完全演化
- 要么真正的 `TurnContext` 语义被更上层/别处消化掉了

这个点在后续 guidebook 里要小心，不要直接照注释说满。

---

## 一句角色定义

如果按源码共读风格压一句话，我会这么写：

> **`build_turns_from_rollout_items(...)` 不是 rollout parser，而是把 persisted rollout stream 归约成客户端 turn history 的公共 replay reducer。**

---

## 为什么这层设计是对的

### 1. live 与 replay 共用同一套 builder
减少语义漂移。

### 2. 显式边界优先，旧格式启发式兼容
兼顾正确性和历史兼容性。

### 3. 允许 late item 回补到旧 turn
比“只看当前 turn”成熟得多。

### 4. compaction-only turn 也保留
说明它在保留语义事件，不只是文本消息。

---

## 继续细拆的话，下一篇最自然的点

最自然继续拆的是：

1. `ThreadHistoryBuilder::handle_event(...)`
2. `handle_user_message(...)`
3. `handle_turn_complete(...)`（builder 版本，不是 app-server 版本）
