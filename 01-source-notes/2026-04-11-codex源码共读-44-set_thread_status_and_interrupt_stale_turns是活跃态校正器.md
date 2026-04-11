---
title: set_thread_status_and_interrupt_stale_turns(...) 是活跃态校正器
date: 2026-04-11
tags:
  - Codex
  - 源码共读
  - app-server
  - thread-status
  - turn-status
source_files:
  - /Users/wangguiping/workspace/codex/codex-rs/app-server/src/codex_message_processor.rs
  - /Users/wangguiping/workspace/codex/codex-rs/app-server/src/thread_status.rs
status: done
---

# Codex 源码共读 44：set_thread_status_and_interrupt_stale_turns(...) 是活跃态校正器

## 这篇看什么

前一篇 `populate_thread_turns(...)` 已经把 turn list 装起来了。

但光有 turns 还不够，因为：
- watch manager 可能有滞后
- 历史里可能还留着 stale `InProgress` turn
- thread 当前是不是 active，不能只靠 rollout/history 直接看出来

这篇只讲：
- `set_thread_status_and_interrupt_stale_turns(...)`

## 先给主结论

如果先留一句话，我会写：

> `set_thread_status_and_interrupt_stale_turns(...)` 的真正角色不是简单给 `thread.status` 赋值，而是在 app-server 的 thread read/resume 视图里完成一次“活跃态校正”：它先用 `resolve_thread_status(...)` 把 watch-manager 的 loaded status 和额外的 live in-progress 证据融合成最终 thread status，再据此把那些在当前 thread 明明不活跃却仍残留为 `InProgress` 的 turn 一律降成 `Interrupted`，从而保证 thread-level status 和 turn-level status 对客户端来说不打架。

再压缩一点：

> **它是 thread/turn 两层状态对齐的最后校正器。**

---

## 第一层：它的工作其实分两半，但必须放在一起

这个函数表面只有两步：
1. 算 thread status
2. 必要时把 stale in-progress turn 改成 interrupted

这两步看起来像能拆开，
但实际上不能。

因为只有先知道：
- 这个 thread 最终算不算 active

你才知道：
- turn 里的 `InProgress` 到底是真活跃，还是脏残留

所以这函数最本质的价值是：

> **把 thread status 计算和 turn status 清洗绑成同一个原子逻辑块。**

这就是它该存在的原因。

---

## 第二层：`resolve_thread_status(...)` 不是普通 helper，而是在修补 watch-state 的滞后窗口

`loaded_status_for_thread(...)` 是 watch manager 给出的状态。

但系统显然知道，这个状态可能滞后。

所以 `resolve_thread_status(...)` 还会额外看：
- `has_live_in_progress_turn`

如果：
- loaded status 是 `Idle` 或 `NotLoaded`
- 但 live in-progress turn 证据存在

它就强行升成：
- `Active`

这说明作者非常清楚一个现实问题：

> **watch-state 不是绝对真相，它可能落后于 listener 已经看到的 live turn 事实。**

所以 `resolve_thread_status(...)` 的本质不是映射函数，
而是 race-window correction。

---

## 第三层：为什么有了这个 correction，还要再扫一遍 turns

因为 thread status 被校正之后，
还要继续回答一个更细的问题：

> 如果最终 thread 不是 active，那 turn list 里还能不能留着 `InProgress`？

答案显然是不应该。

所以函数会继续：
- 如果最终 status 不是 `Active`
- 就把所有 `TurnStatus::InProgress` 统一改成 `Interrupted`

这一步是在做：

> **turn 列表和 thread 状态的语义收口。**

否则客户端就会看到非常别扭的组合：
- thread status = Idle / NotLoaded / SystemError
- turn status = InProgress

这对 UI 和用户心智都很糟糕。

---

## 第四层：它显式承认 persisted history 不一定和当前 live runtime 完全一致

这一点非常重要。

如果 persisted history 永远绝对可信，
就不需要这个函数。

有它就说明系统明确承认：
- rollout/history 可能滞后
- watch manager 也可能滞后
- active turn snapshot 可能更接近真相

所以 app-server 返回给客户端的 thread 视图，
并不是机械照抄任何一层存储，
而是：

> **基于多路状态源做最终一致性修正后的视图。**

而这个函数就是最后那道修正闸门。

---

## 第五层：它对 `SystemError` 的态度，说明“明确错误”优先级高于“推测活跃”

`resolve_thread_status(...)` 只会把：
- `Idle`
- `NotLoaded`

在有 live evidence 时升成 `Active`。

它不会把：
- `SystemError`

升成 `Active`。

这说明作者在状态优先级上做了一个很明确的选择：

> **明确的系统错误 > 推测出来的活跃态。**

这个选择我觉得是对的。

因为如果系统已经进入 error 态，再强行“看起来活跃”，
对用户反而更误导。

---

## 第六层：active flags 的处理也透露出它的边界

如果 watch manager 原本已经有：
- `Active { active_flags }`

那函数会保留这些 flags。

如果它只是因为：
- `has_live_in_progress_turn == true`
- 而把 `Idle` / `NotLoaded` 升成 `Active`

那会得到：
- `Active { active_flags: [] }`

这说明这个函数只想恢复：
- **活跃性**

而不是伪造更多它根本不知道的子状态。

这也很克制。

就是说它只修自己确信能修的那一层，
不会顺手捏造“在等 approval / 在等 user input”这种更细的 active substate。

---

## 第七层：它的真正角色不是状态推导，而是状态一致性修复

如果把它理解成“状态推导函数”，会低估它。

因为真正难点不是推出一个 status，
而是：
- `thread.status`
- `thread.turns[*].status`

两层不要互相打脸。

所以我更愿意把它定义成：

> **状态一致性修复函数**

或者更直白一点：

> **活跃态校正器**

这个名字比“set status”更贴近真实职责。

---

## 一句角色定义

如果按源码共读风格压一句话，我会这么写：

> **`set_thread_status_and_interrupt_stale_turns(...)` 的核心不是给 thread 填状态，而是把 watch-state、live evidence 和 turn list 重新校正到一个对客户端更一致的活跃态视图上。**

---

## 为什么这层设计是对的

### 1. 明确承认多路状态源并不天然一致
这是系统成熟的标志。

### 2. 先修 thread，再修 turn
顺序逻辑清楚。

### 3. 不伪造不知道的 active flags
修正很克制，不乱补细节。

### 4. 用 `Interrupted` 吃掉 stale `InProgress`
比继续留脏状态更合理。

---

## 继续细拆的话，下一篇最自然的点

最自然继续拆的是：

1. `resolve_thread_status(...)`
2. `ThreadWatchManager::loaded_status_for_thread(...)`
3. `loaded_thread_status(...)`

因为这三块正好构成它上游的状态推导链。
