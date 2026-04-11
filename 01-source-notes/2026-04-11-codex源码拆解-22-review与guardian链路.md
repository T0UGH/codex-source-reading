# Codex 源码拆解 22：review 与 guardian 链路

## 这篇看什么

回答：**用户显式触发的 review，和审批场景里的 guardian review，到底是什么关系？inline / detached / guardian trunk/fork 又分别在干什么？**

## 先给结论

这是两条相关但不同的链：

1. `/review`
   - 是**用户显式发起的审查工作流**
   - 可 inline，也可 detached review thread
2. `guardian`
   - 是**审批/安全场景里的 reviewer 子系统**
   - 不等于普通 review
   - 有 trunk session + ephemeral fork 复用策略

一句话：

> review 是产品功能；guardian 是审批 reviewer 基础设施。

## 关键代码路径

- `codex-rs/app-server/src/codex_message_processor.rs`
- `codex-rs/core/src/codex.rs`
- `codex-rs/core/src/tasks/review.rs`
- `codex-rs/core/src/guardian/review.rs`
- `codex-rs/core/src/guardian/review_session.rs`
- `codex-rs/core/src/guardian/prompt.rs`
- `codex-rs/app-server/src/bespoke_event_handling.rs`

## `/review` 主链

### 从 app-server 进来
`review_start` 会：
- 解析 target
- 决定 delivery mode
- 走 inline 或 detached

### inline review
- 仍在原 thread 上执行
- 返回的 `review_thread_id` 其实就是原 thread id
- 通过 `Op::Review` 进入 core

### detached review
- 先从 parent rollout fork 一个新 thread
- fork snapshot 用的是 `Interrupted`
- 再在新 thread 上提交 `Op::Review`
- app-server 还会给客户端先发 `ThreadStarted`

所以 detached review 的本质是：

> 为 review 结果单独开一个 thread surface，而不是把 review 硬塞在主 thread 里。

## core 里的 review 不是普通 user turn

`tasks/review.rs` 很明确：

- review conversation 是一个 locked-down reviewer subagent
- `approval_policy = never`
- base instructions 是专门的 `REVIEW_PROMPT`
- `SubAgentSource::Review`

这说明 review 运行时语义是：
- 特化子 agent
- 一次性 reviewer conversation
- 结果再 materialize 回主系统

所以不要把 review 理解成“普通 prompt + 特别模板”那么浅。

## guardian 链路什么时候触发

只有当：
- `approval_policy == on-request`
- `approvals_reviewer == guardian_subagent`

才会把某些 approval request 路由给 guardian reviewer。

触发点包括：
- shell/exec approval
- patch approval
- network approval
- 一些 delegated 场景

所以 guardian 不是通用审查模式，而是：

> 针对 approval / review request 的专门 reviewer backend。

## guardian 和普通 review 最大的差异

### 普通 review
- 面向用户显式功能
- 可 inline/detached
- 每次围绕具体 review request 起 reviewer 流程

### guardian
- 面向审批判断
- 会发 `GuardianAssessment` 生命周期事件
- 重点不是产出“review 文本线程”，而是产出审批判断与审查轨迹

## guardian session manager 很像一个 reviewer 池

这是本次最值的点之一。

`guardian/review_session.rs` 里能看到：

- 有 trunk review session
- trunk 空闲且 config-compatible → 复用
- trunk 忙 → fork ephemeral review
- trunk config 不兼容且空闲 → 替换 trunk
- trunk 可记录 `last_committed_fork_snapshot`

这说明 guardian 不是每次都冷启动 reviewer，而是在做：

> 可复用的 reviewer session 基础设施。

这么做的好处：
- 保留 reviewer context
- 减少重复启动成本
- 同时允许并发审批通过 fork 隔离

## guardian prompt 也有增量策略

不是每次都塞全量 transcript。

当前能看到：
- 第一轮可吃完整 transcript
- 后续可走 transcript delta
- 恰好有一次 prior review 时还会追加 follow-up reminder

这说明 guardian 已经在认真优化：
- reviewer 的上下文连续性
- token 成本
- follow-up 审查体验

## app-server 如何把 guardian 事件投影给客户端

core 会发：
- `EventMsg::GuardianAssessment(...)`

app-server 再：
- 产出 guardian approval review started/completed 通知
- 必要时还会 synthesize command-execution items

所以客户端看到的 guardian UX，其实是 app-server 在 `bespoke_event_handling` 里做了一层协议级再表达。

## detached review fork vs guardian ephemeral fork

这两个很容易混。

### detached review fork
- 是用户 review 功能的 thread-level 分叉
- 来源是 parent thread rollout
- 目的是给 review 一个单独 thread

### guardian ephemeral fork
- 是 reviewer 基础设施里的 session-level fork
- 来源是 guardian trunk 的 committed snapshot
- 目的是 reviewer 并发与复用

虽然都叫 fork，但产品语义完全不同。

## 当前边界判断

- `/review`：用户显式审查功能
- inline review：原 thread 上执行
- detached review：另开 review thread
- guardian：approval reviewer 子系统
- guardian trunk：可复用 reviewer 主干 session
- guardian ephemeral fork：并发/隔离用的 reviewer 分支

## 设计判断

### 1. `/review` 和 guardian 不能混为一谈
它们有共享 reviewer 技术味道，但产品定位不同。

### 2. guardian session manager 是这套系统里很高级的一层
它把 reviewer 从“一次性任务”升级成了“可复用基础设施”。

### 3. detached review 是对用户体验友好的 thread packaging
这一步让 review 结果可以独立存在、独立跟踪、独立恢复。

## 还值得继续观察的点

1. guardian analytics 发射点还值得确认
2. `thread_history` 为什么对某些 guardian approved 场景不再 synthesize command item，后面可补一篇
3. review / guardian 是否最终会在更高层统一成一个 reviewer framework
