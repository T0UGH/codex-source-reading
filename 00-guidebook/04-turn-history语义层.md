---
title: Codex guidebook 04｜turn-history 语义层
date: 2026-04-12
status: draft
---

# 04｜turn-history 语义层

## 先给结论

如果说上一章讲的是 app-server 如何把 live runtime 投影成 thread 控制面，那么这一章讲的就是：

> **Codex 如何把 event stream 重新组织成可展示、可恢复、可继续推进的 turn history。**

这里最关键的不是“把日志读出来”，而是“把事件解释成一个个 turn”。

从源码上看，这套语义层主要由两部分构成：

1. **`ThreadHistoryBuilder`**：负责把 live/persisted events 归约成 turn 语义
2. **rollout reconstruction**：负责在恢复、rollback、compaction 等场景下，把持久化历史重建成正确的会话语义

所以这章的核心结论可以先立住：

> **turn history 不是 event log 的镜像，而是一个 semantic projection。**

---

## 1. 为什么必须有一层 turn 语义归约

如果系统只是把所有 event 原样堆出来，会有几个问题：

- 用户很难把一串事件理解成“一个交互轮次”
- rollback / compaction 之后，历史不再是简单 append-only
- running thread 的当前 turn 和 persisted history 需要合并
- 旧协议流和新协议流可能并不完全一致

所以系统必须回答的问题不是“发生了哪些 event”，而是：

- 哪些 event 属于同一个 turn
- 什么时候一个 turn 开始
- 什么时候一个 turn 结束
- 中断、错误、rollback、compaction 之后，这个 turn 该怎么表示

这就是 turn-history 语义层存在的原因。

---

## 2. `ThreadHistoryBuilder` 是这层的核心 reducer

`build_turns_from_rollout_items()` 本身并不复杂，它做的事情很直接：

- 创建一个 `ThreadHistoryBuilder`
- 把 rollout items 逐个喂进去
- 最后 `finish()`

这说明设计者把真正的 turn 语义，集中在 builder 上，而不是散在各处。

因此，`ThreadHistoryBuilder` 最准确的定位是：

> **turn 语义归约器。**

而且它不是只服务一种场景。它同时被用于：

- 从 rollout replay 历史
- 追踪当前 live turn
- 为 running-thread resume 提供 active turn snapshot

同一个 builder 同时服务 persisted history 和 live state，这一点非常关键。它说明：

> **系统不想维护两套 turn 语义，而是想用同一套 reducer，处理两种来源。**

---

## 3. turn 的边界：显式优先，兼容旧流

从 builder 的逻辑看，Codex 对 turn 的态度是：

> **显式 turn 生命周期优先，但仍兼容旧流里的隐式边界。**

新的事件流会明确告诉系统：

- turn started
- turn complete
- turn aborted

这时 builder 会：

- 在 `TurnStarted` 时关闭旧 turn、打开新 turn
- 记录 turn_id 和 started_at
- 在 `TurnComplete` / `TurnAborted` 时优先按 turn_id 封口

但系统没有假设所有历史流都这么完美。对于没有完整显式生命周期的旧流，它还保留了一套隐式规则，其中最关键的一条就是：

- **用户消息仍然是 turn 划分的重要边界。**

这就是为什么 `handle_user_message(...)` 这么关键。

---

## 4. `handle_user_message` 其实在决定“一个新 turn 从哪开始”

这不是一个普通的 append 函数，而是 turn 语义层里最重要的边界函数之一。

它负责的核心问题是：

- 当前用户消息，是不是应该开启一个新的 turn
- 如果旧流没有显式边界，要不要先把前一个隐式 turn 收口
- 哪些特殊 turn 不能简单丢掉

也就是说，它决定的不是“怎么记一条用户输入”，而是：

> **用户输入如何成为 turn 划分规则的一部分。**

这也是为什么后面 rollback、resume、history replay 都要围绕 user-turn 语义来做，而不是仅围绕 event arrival order 来做。

---

## 5. turn-history 不是简单按到达顺序堆叠

这是这层最重要的性质之一。

在很多地方，builder 都不是“来了什么就 append 在最后”。它会结合：

- turn_id
- active turn
- 历史 turn 匹配
- 补偿逻辑

来决定一个 item 最终属于哪个 turn。

这意味着：

- 历史不是 arrival-order first
- 历史更接近 semantic-association first

一个典型例子是某些 exec end / tool end 之类的事件。它们有可能到得比较晚，甚至和实时顺序并不完全一致，但 builder 仍然会尽量把它们挂回正确的 turn，而不是机械地丢进“最新 turn”。

所以可以明确一个判断：

> **turn-history 是按语义重建的，不是按事件到达顺序简单堆出来的。**

---

## 6. 为什么 compaction-only turn 不能一概丢掉

如果只看直觉，空 turn 似乎没必要保存。但 builder 的逻辑里有一个关键例外：

- **compaction-only turn 不能简单当成空 turn 丢掉。**

原因不难理解。

有些 turn 即使没有留下很丰富的对话内容，它仍然承载了历史形状的变化——特别是 compaction 之后，历史基线可能已经变了。如果把这些 turn 一律丢掉，后面的恢复和 history 语义就会被悄悄改写。

这也说明 builder 不是只在乎“显示出来有没有内容”，它还在维护：

> **历史结构的正确性。**

所以对 turn-history 来说，“是否为空”和“是否语义上重要”不是一回事。

---

## 7. `active_turn_snapshot` 为什么是一个语义接口，而不是简单 getter

`active_turn_snapshot()` 乍看像是“拿当前 turn”的函数，但从行为上看，它更像一个语义接口。

它的返回不是机械的“当前活动 turn 或空”，而更接近：

- 如果当前有 in-progress turn，就返回它
- 如果没有，也可能返回最近一个适合作为视图基础的 turn

这点很重要，因为它解释了为什么这个接口能服务：

- live UI 通知
- running-thread resume 时的活跃态叠加
- 某些 turn 视图装配场景

所以更准确的说法不是“当前活动 turn getter”，而是：

> **当前/最近 turn 的对外投影口。**

这也解释了为什么它必须和 `has_active_turn()` 之类的判断一起读，不能单独理解。

---

## 8. `ThreadHistoryBuilder` 管的是“turn 语义”，rollout reconstruction 管的是“历史语义”

如果只看 builder，很容易觉得 turn-history 语义层已经完整了。但真正把系统讲完整，还要加上 rollout reconstruction。

这部分做的事情更偏“历史级”的重建，而不是“当前 turn 级”的归约。它要处理的包括：

- replacement-history compaction
- previous turn settings
- reference context item
- rollback 后哪些 user turns 应该被丢掉
- surviving suffix 应该如何 forward replay

这意味着两层语义在这里是分开的：

### `ThreadHistoryBuilder`
负责：
- event 到 turn 的语义归约
- 当前 turn 视图
- 基础 replay turn slicing

### rollout reconstruction
负责：
- persistence 层的历史修复
- rollback/compaction 后的 surviving history 语义
- 从持久化历史中恢复真正还能继续工作的上下文

所以更完整的判断应该是：

> **builder 定义“turn-shaped semantics”，reconstruction 定义“history-shaped semantics”。**

---

## 9. rollback 不是“删几个 event”，而是“删几个 user turns”

这是 rollout reconstruction 很能体现设计意图的地方。

rollback 的语义并不是：

- 删除最近 N 个 event

而更接近：

- **删除最近 N 个 user turns**

这两者差别很大。因为一个 user turn 里可能包含很多 event，也可能和一些 non-user spans 交错。如果只按 event 删除，历史会变形；而按 user-turn 语义删，历史才能保持真正的对话结构。

这说明系统真正想保住的，不是原始事件流，而是：

> **用户视角 / session 视角里的会话轮次结构。**

这也是为什么 reconstruction 需要 reverse replay + forward replay 两阶段：先判断保留哪些语义片段，再把它们重新 materialize 成可继续运行的 history。

---

## 10. running thread 的可见 history 为什么要“持久化历史 + live turn overlay”

这点把前一章和这一章真正接上了。

对于一个还在运行的 thread，系统不能只把 rollout 里的历史给客户端看。因为：

- rollout 代表的是已持久化历史
- 当前正在进行中的 turn，可能还没完全写进去

所以更合理的做法是：

1. 先用 rollout items rebuild 出 persisted turns
2. 再把 `active_turn_snapshot()` 覆盖上去
3. 最后再修正 status 和 stale in-progress 语义

这说明 turn-history 在 Codex 里不是纯历史仓，而是：

> **一个同时服务 replay 与 live projection 的语义层。**

---

## 11. 这一章最重要的判断

读完这章，应该稳定这些结论。

### 判断 1：turn-history 不是 event log 镜像
它是 semantic projection。

### 判断 2：`ThreadHistoryBuilder` 是 turn 语义权威
persisted history 和 live turn 都尽量经过同一套 reducer。

### 判断 3：显式 turn 生命周期优先，但系统兼容旧流
旧流里 user message 仍然承担重要边界作用。

### 判断 4：`handle_user_message` 是 turn 边界函数
它决定隐式 turn 如何收口和重开。

### 判断 5：item 归属按语义，不只按到达顺序
turn_id 和 active-turn 语义优先于 arrival order。

### 判断 6：compaction-only turn 不能简单丢弃
因为它可能承载历史结构变化。

### 判断 7：`active_turn_snapshot` 是对外投影口
不是一个朴素 getter。

### 判断 8：builder 与 reconstruction 分工不同
前者管 turn semantics，后者管 persisted history semantics。

### 判断 9：rollback 语义是删 user turns，不是删 event
这保证恢复后的会话结构仍然成立。

---

## 12. 下一章该接什么

turn-history 语义说清以后，下一步最自然的问题就是：

> **Codex 如何把执行过程本身做成一个会话化、可恢复、可观察的子系统？**

也就是：

- 为什么 `exec.rs` 和 `unified_exec` 不是一回事
- 为什么 unified-exec 不是简单包一层 shell
- output / transcript / end event / process store 是怎么闭环的

也就是下一章的主题：**unified-exec 执行子系统。**

---

## 相关阅读

- 想回看这条语义层挂在哪条控制面主线里：[[03-app-server与thread-turn主线]]
- 想继续看执行系统如何把输出和 turn 语义接起来：[[05-unified-exec执行子系统]]
- 想直接定位 turn 语义相关关键函数：[[11-关键函数索引]]、[[12-调用链索引]]

## 关键文件

- `codex-rs/app-server-protocol/src/protocol/thread_history.rs`
- `codex-rs/app-server/src/thread_state.rs`
- `codex-rs/app-server/src/codex_message_processor.rs`
- `codex-rs/core/src/thread_manager.rs`
- `codex-rs/core/src/codex/rollout_reconstruction.rs`
