---
title: Codex guidebook 15｜guardian analytics 为何还没真正接上
date: 2026-04-12
status: draft
purpose: micro-topic
---

# 15｜guardian analytics 为何还没真正接上

## 先给结论

这也是一个很值得单独拿出来打死的微缺口：

> **guardian review 在 runtime 和 app-server/UI 层已经是可观察系统，但在 analytics 链上，目前看起来还没有真正接线。**

更准确地说：

- analytics schema 有
- analytics reducer 有
- analytics client API 也有
- 但在 guardian runtime 主链里，我没有看到真正发射 guardian analytics fact 的地方

所以当前最稳的判断不是“有问题但不知道在哪”，而是：

> **guardian analytics 现在更像 analytics scaffolding 已经搭好，但 runtime emission 还没有补齐。**

---

## 1. 先把两个“可观察”分开

如果不把这一步拆开，很容易误判成“guardian analytics 已经有了”。

这里其实有两种完全不同的可观察性：

### 第一层：runtime / app-server / UI 可观察
这条链已经存在。

也就是说，guardian review 发生时，系统已经会发出：

- `GuardianAssessment(InProgress)`
- `GuardianAssessment(Approved/Denied/TimedOut/Aborted)`
- app-server 侧的 guardian review notifications
- 某些场景下的 command execution decline/fail 投影

所以如果只看 UI 或 app-server 协议面，你会觉得 guardian 已经是一个正式可观察子系统。

### 第二层：analytics pipeline 可观察
这条链要求的是：

- 真的调用 analytics client
- 真的生成 analytics fact
- 真的走 reducer
- 最终产出 `codex_guardian_review` 之类事件

而这一层，目前看起来**并没有真正接上 runtime 主链**。

所以这篇要说清楚的，不是 guardian 有没有事件，而是：

> **guardian 有没有接上 analytics。**

---

## 2. analytics 这边其实已经把架子搭得很完整了

先看 analytics crate，本身并不缺这套东西。

目前已经明确存在：

### 1) client API
`AnalyticsClient::track_guardian_review(...)`

这说明设计者早就预留了 guardian review 的埋点入口。

### 2) fact 类型
`CustomAnalyticsFact::GuardianReview(...)`

这说明 guardian review 已经被纳入 analytics 内部事实模型，而不是靠日志硬拼。

### 3) 事件 schema
`GuardianReviewEventParams`

这里面已经定义了不少字段，比如：

- request source
- reviewed action
- outcome / rationale
- guardian thread id
- guardian session kind
- guardian model / reasoning effort
- timeout / token / latency 等指标

这说明它不是“顺手记个日志”的级别，而是已经按正式事件来设计了。

### 4) reducer
analytics reducer 已经能把这类 fact 降成最终 event，比如 `codex_guardian_review`。

所以如果只看 analytics 子系统，会很自然地以为：

> **guardian analytics 应该已经接好了。**

但问题恰恰在 runtime 侧。

---

## 3. 真正的问题：我没找到 runtime emission

继续往 runtime 主链里追，就会发现一个很关键的断点：

- `track_guardian_review(...)` 的定义在 analytics 里
- reducer 也在 analytics 里
- 但我没有在 guardian runtime 主链里找到实际调用点

也就是说，现在的状态更像这样：

- analytics 已经准备好接收 guardian review 事件
- 但 guardian runtime 还没有把这些事件真正送过去

这不是“埋点不够丰富”，而是更基础的问题：

> **当前看起来根本还没真正发。**

这个判断已经比之前“似乎没接好”更明确了。

---

## 4. guardian runtime 现在真正发的是什么

guardian runtime 并不是什么都没发。相反，它在自己的运行链里会明确发：

- `EventMsg::GuardianAssessment(InProgress)`
- `EventMsg::GuardianAssessment(Aborted)`
- `EventMsg::GuardianAssessment(Approved/Denied)`

这些事件会继续被：

- app-server 的 `bespoke_event_handling` 消化
- 转成 app-server 通知 / protocol payload
- 最后进入 UI 可见层

所以 guardian 不是没有事件，而是：

> **它的事件现在主要停留在 protocol/UI 世界，而不是 analytics 世界。**

这就是最核心的断裂点。

---

## 5. app-server 已经会把 guardian review 投影给 UI

这一点也值得单独写清楚，因为它正是最容易误导人的地方。

app-server 里已经有明确逻辑把 `GuardianAssessment` 变成：

- started / completed 这类 guardian review notifications
- 携带 review status、risk level、authorization、rationale 等字段
- 某些命令类场景还会转成 command execution decline/fail 的 item 投影

所以从产品观感上，guardian review 已经不是“后台逻辑”，而是一个：

> **有正式 protocol surface 的审查系统。**

这就是为什么很多人会下意识认为 analytics 也该已经有了。

但实际上，当前链路只走到了 protocol / UI 这一层，并没有继续沉到底下的 analytics fact emission。

---

## 6. analytics reducer 为什么帮不了这个问题

还有一个很容易误解的点是：

“既然 app-server 已经有 guardian review 通知，analytics reducer 不能直接从 notification 里推出来吗？”

目前看，答案是不行。

原因在于 analytics reducer 对 notification 的处理非常保守。它不会因为收到 protocol notification 就自动把 guardian review 还原成 analytics fact。

也就是说：

- guardian protocol event ≠ guardian analytics fact
- reducer 不会自动替你补 runtime emission

这说明这条缺口不是“还差一点 mapping”，而是：

> **guardian runtime 侧本来就还没把 analytics fact 发出来。**

---

## 7. 为什么这更像“未接线”，不是“有意不打点”

如果这真是刻意不打点，我更期待看到几种证据之一：

- 明确注释说 guardian review 不进 analytics
- schema 只做半套，不做完整 reducer/client API
- 或 runtime 里有显式 no-op / disable 路径

但现在看到的反而是相反信号：

- client API 有
- fact 类型有
- schema 很完整
- reducer 也有

这套架子搭得这么完整，说明方向很明确：

> **设计上是想把 guardian review 纳入 analytics 的。**

因此，现在最合理的结论不是“故意没有”，而是：

> **实现还没接到 runtime 主链。**

---

## 8. 这会带来什么实际后果

这个缺口的影响，不在 UI 上，而在系统观测层。

### 现在已经能做的
- 用户能看到 guardian review 生命周期
- app-server / UI 能看到状态、结果、rationale
- approval 基础设施本身能工作

### 现在做不到或至少看起来没做的
- 在 analytics 里稳定统计 guardian review
- 把 guardian review 纳入正式埋点事件流
- 做更完整的跨 session / 跨 runtime 行为分析

所以这个问题不是“功能没法用”，而是：

> **guardian 已经是运行系统，但还没完全成为 analytics 系统的一部分。**

---

## 9. 当前最稳的判断

可以把这件事压成下面 4 条。

### 判断 1：guardian runtime 是有正式事件流的
不是黑箱逻辑。

### 判断 2：guardian protocol / UI 投影已经存在
started/completed/status/rationale 都有。

### 判断 3：guardian analytics schema / reducer / client API 也都存在
analytics 侧架子不是空的。

### 判断 4：真正缺的是 runtime emission
所以现在更像“埋点接线未完成”，而不是“根本没设计过”。

---

## 10. 如果继续补，这一刀该怎么下

这条线已经不需要再问“有没有问题”，而该直接问：

> **guardian runtime 到底应该在哪个边界发 `track_guardian_review(...)`？**

继续建议：
- 从 `review_approval_request(...)` / `run_guardian_review(...)` 这一侧下手
- 看最合理的发射时机是：
  - terminal decision 时
  - timeout/abort 时
  - 还是 started + terminal 两段式
- 再对齐 analytics schema 里那些已经定义好的字段

如果还要继续补一个最小闭环，这就是下一刀。

---

## 11. 这一篇的最终结论

一句话收口：

> **guardian review 现在已经是 protocol/UI 上成熟的可观察系统，但在 analytics 层，仍然停留在“schema 和 reducer 已经搭好，runtime emission 还没真正接上”的状态。**

这不是推测，是当前源码结构最稳的读法。

---

## 关键文件

- `codex-rs/analytics/src/client.rs`
- `codex-rs/analytics/src/events.rs`
- `codex-rs/analytics/src/facts.rs`
- `codex-rs/analytics/src/reducer.rs`
- `codex-rs/core/src/guardian/review.rs`
- `codex-rs/core/src/tools/runtimes/unified_exec.rs`
- `codex-rs/core/src/tools/network_approval.rs`
- `codex-rs/core/src/mcp_tool_call.rs`
- `codex-rs/core/src/tools/orchestrator.rs`
- `codex-rs/core/src/codex_delegate.rs`
- `codex-rs/app-server/src/bespoke_event_handling.rs`
