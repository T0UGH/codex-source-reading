---
title: handle_user_message(...) 是 implicit turn 收口与新输入归属器
date: 2026-04-11
tags:
  - Codex
  - 源码共读
  - thread-history
  - user-message
  - implicit-turn
source_files:
  - /Users/wangguiping/workspace/codex/codex-rs/app-server-protocol/src/protocol/thread_history.rs
status: done
---

# Codex 源码共读 58：handle_user_message(...) 是 implicit turn 收口与新输入归属器

## 这篇看什么

前面已经看过：
- `handle_event(...)` 是总分发器
- `active_turn_snapshot(...)` 是当前 turn 的对外投影口

那在具体 helper 里，最值得单独拆的一个就是：
- `handle_user_message(...)`

因为 turn 划分是否稳定，很多时候就靠它这一步。

## 先给主结论

如果先留一句话，我会写：

> `handle_user_message(...)` 的本质不是往当前 turn 里 append 一条 user item，而是 thread-history reducer 里负责 implicit turn 收口和新输入归属的边界函数：如果当前 turn 不是显式打开的，并且也不是“只有 compaction 标记的空壳 turn”，它会先把这个旧的 implicit turn finalize 掉；然后再复用现有 turn 或新建 turn，把新的 `UserMessage` 作为 fresh turn input 落进去。也就是说，这里真正处理的是旧格式兼容下的 turn 划分修正，而不是简单记消息。

再压缩一点：

> **用户消息不是普通 item，它还是 implicit turn 的切分信号。**

---

## 第一层：这段逻辑最重要的不是 push message，而是前面的条件性封口

函数开头就是：
- 如果当前 turn 存在
- 且 `!opened_explicitly`
- 且不是 `saw_compaction && items.is_empty()`
- 那就 `finish_current_turn()`

这三行比后面的 push item 更重要。

因为它在表达：

> **新的用户输入默认不应该落进旧的 implicit turn。**

这就是 backward compatibility 下的 turn 修正规则。

---

## 第二层：显式 turn 优先，implicit turn 只是兼容兜底

注释已经很明确：
- User messages should stay in explicitly opened turns.
- For backward compatibility with older streams that did not open turns explicitly...

也就是说作者的理想模型是：
- turn 由 `TurnStarted/TurnComplete` 这些显式边界管理

但现实里旧流不一定有，
所以才让 `UserMessage` 兼任一个：
- implicit turn boundary heuristic

这再次验证了前面的判断：

> **Codex 的 thread-history 逻辑一直是“显式边界优先，启发式兼容兜底”。**

---

## 第三层：compaction-only 空壳 turn 是故意豁免的

这个条件很值：
- `!(turn.saw_compaction && turn.items.is_empty())`

也就是说，如果当前 turn 只有 compaction 标记、没有普通 items，
新用户消息到来时不会先把它粗暴 finalize 掉再另起。

这说明作者知道这类 turn 不是普通 implicit 噪声，
而是有特殊语义的保留壳。

所以这里不是简单的“implicit 就关”，
而是：

> **普通 implicit turn 要收口；带 compaction 语义的空 turn 要被特殊保留。**

这很细，但非常像真实线上兼容逻辑。

---

## 第四层：它 `take current_turn` 再放回去，说明这里是在安全地重建所有权边界

后半段不是直接 `ensure_turn().items.push(...)`，
而是：
- `let mut turn = self.current_turn.take().unwrap_or_else(...)`
- push user item
- `self.current_turn = Some(turn)`

这说明它希望在完成“必要封口判断”之后，
显式地拿到一个将被承接新输入的 turn 实体。

这个写法的好处是：
- 前半段可能已经 `finish_current_turn()` 改写了状态
- 后半段重新明确当前消息落在哪个 turn 上

所以它不是语法偏好，
而是在做 turn ownership 的重新决策。

---

## 第五层：用户消息 item id 走的是 builder 自增序列，不是外部事件 id

它这里用的是：
- `let id = self.next_item_id()`

这说明 user message 在 v2 turn item 里，
并不是直接拿外部 event 自带的某个稳定 id，
而是由 builder 在重建 turn 视图时分配顺序 item id。

所以它更像：
- **渲染/协议层 item identity**
- 而不是底层 event identity

这也说明 thread-history 本来就不是 event log 直接镜像。

---

## 第六层：`build_user_inputs(...)` 说明 user message 在这里已经是多模态输入聚合点

后面 push 进去的不是单纯字符串，
而是 `build_user_inputs(payload)` 生成的 `Vec<UserInput>`：
- text
- remote images
- local images

所以 `handle_user_message(...)` 也不是只处理聊天文本，
而是在 turn 语义层负责：

> **把一次用户输入事件折叠成统一的多模态输入 item。**

这对后面 guidebook 讲协议 shape 很重要。

---

## 第七层：这又是一个“很小但决定 turn 切分质量”的函数

源码里很多大判断，其实不是在大函数里做的。

`handle_user_message(...)` 就是典型：
- 很短
- 但直接决定老格式回放时 turn 会不会串台
- 还要兼顾 compaction 特例
- 还要把输入转成统一 `UserInput` 列表

这就是我前面一直在说的：

> **Codex 的核心质量很多来自这些边界极准的小函数。**

---

## 一句角色定义

如果按源码共读风格压一句话，我会这么写：

> **`handle_user_message(...)` 是 thread-history reducer 里负责 implicit turn 收口和新输入归属的边界函数：它把新用户输入同时当成内容写入点和旧格式兼容下的 turn 切分信号。**

---

## 为什么这层设计是对的

### 1. 显式 turn 优先，旧格式靠 user message 兜底切分
兼容性和正确性兼顾。

### 2. compaction-only turn 有特例保护
不丢语义壳。

### 3. 输入被统一映射成 `UserInput` 列表
协议更干净。

### 4. turn 所有权切换清楚
不容易把新消息塞进旧 turn。

---

## 继续细拆的话，下一篇最自然的点

最自然继续拆的是：

1. `build_user_inputs(...)`
2. `upsert_item_in_turn_id(...)`
3. `HeadTailBuffer`
