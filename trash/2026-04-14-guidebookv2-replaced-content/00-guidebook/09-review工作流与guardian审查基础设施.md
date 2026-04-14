---
title: Codex guidebook 09｜review 工作流与 guardian 审查基础设施
date: 2026-04-12
status: draft
---

# 09｜review 工作流与 guardian 审查基础设施

## 先给结论

`/review` 和 guardian 很容易被误读成同一件事：都是“让另一个 agent 帮你审一下”。

但从源码职责看，它们属于两层完全不同的系统：

> **`/review` 是面向用户的 review 工作流，guardian 是面向 approval 的审查基础设施。**

前者的目标是：
- 产出 review 结果
- 让用户看到一份结构化审查内容

后者的目标是：
- 在需要批准时替用户做审查判断
- 决定该批准、拒绝、超时还是中止
- 把这个判断反馈回执行链

所以这章最重要的事，不是再说一遍“它们不一样”，而是把两条运行链讲清楚。

---

## 1. `/review`：一个显式的用户工作流

先看 `/review`。

从实现上看，它不是一个底层 approval 机制，而是一个明确的 task-mode 工作流。它会：

- 启动一个专门的 review conversation
- 用受限配置跑一条 review 子线程
- 处理事件流
- 最后产出 review output，并以用户可见的形式退出 review mode

这说明 `/review` 的定位很明确：

> **它是为“生成 review 内容”而设计的产品工作流。**

这条链的重点不是授权，而是内容产出。

从配置上也能看出来，它会主动收紧很多能力：

- 关闭 web search
- 关闭 spawn/collab
- 固定 review 相关 base instructions
- 把 `approval_policy` 设成 `never`

所以 `/review` 不是“让一个强力 agent 自由发挥地检查”，而是一条约束很清楚的 reviewer workflow。

---

## 2. guardian：不是给用户看的工作流，而是 approval infrastructure

guardian 的位置完全不一样。

它不是一个“用户点一下就看 review 结果”的显式模式，而是嵌在 approval routing 里的基础设施。它回答的问题是：

> **当系统遇到需要批准的动作时，是不是可以不用把决定抛给用户，而由一个受约束的 reviewer subagent 先做判断？**

所以 guardian 更接近：

- privileged action reviewer
- approval gate infrastructure
- 审批控制链上的一层自动评估器

这也是为什么 guardian 不只挂在一个地方，而是会插到多种 approval surface 上：

- tool approvals
- network approval
- MCP tool approvals
- patch approvals
- delegated subagent 的 approval 回流

所以 guardian 的定位应该是：

> **审查基础设施，而不是单独一个用户工作流。**

---

## 3. guardian 什么时候会被激活

guardian 不会无条件介入。它有明确的路由门槛。

系统只有在这些条件成立时，才会把 approval 路由给 guardian：

- `approval_policy == OnRequest`
- 当前 turn 配置里 `approvals_reviewer == GuardianSubagent`

也就是说，它不是默认审批系统，而是：

- 在需要 ask-for-approval 的模式下
- 又显式启用了 guardian reviewer
- 才会接管这条审批链

这个门槛很重要，因为它说明 guardian 不是“全局后台守卫”，而是：

> **可配置的审批替代路径。**

---

## 4. guardian 真正做的事：不是执行，而是裁决

guardian 的主链里，最重要的一个判断是：它的职责不是代替工具执行，而是给出审查结果。

一条典型链路会是：

1. 某个动作触发 approval 需求
2. guardian review 被启动
3. 系统立即发一个 `GuardianAssessment(InProgress)`
4. guardian session 在受限环境里运行审查
5. 最后给出：
   - Approved
   - Denied
   - TimedOut
   - Aborted
6. 结果被反馈回原执行链

也就是说，guardian 的输出不是“做完动作”，而是：

> **关于这个动作该不该被允许的判断。**

这一点和 `/review` 的“产出 review 内容”本质不同。

---

## 5. guardian 为什么是 fail-closed

从错误处理方式看，guardian 的设计态度非常明确：

- review 出错 → 倾向 deny
- review 超时 → 倾向 deny
- 被取消 / 中止 → 返回 abort

这说明它在安全边界上采用的是典型的 fail-closed 思路。

也就是说，如果 guardian 自己不能稳定给出可信结论，系统不会乐观地继续执行，而是倾向收紧。

这也进一步证明：

> guardian 是 approval control plane 的一环，不是普通辅助 agent。

因为只有在“负责授权”的语境里，fail-closed 才是自然默认值。

---

## 6. guardian session 为什么要有 trunk 和 ephemeral fork

guardian 不是每次审批都临时 new 一个完全无记忆的 reviewer。它有一套更复杂的会话管理方式：

- 一个可复用的 trunk session
- 若干临时 ephemeral review sessions

这套设计背后的意图很明显：

### trunk session
- 保存可复用的上下文
- 在条件一致时复用已有 reviewer 会话
- 降低每次审查都从零启动的成本

### ephemeral session
- 当 trunk 正在忙
- 或复用 key 不匹配
- 或需要 fork 一个临时审查分支

系统会开一个临时 review session 来完成这次审批。

这说明 guardian 不是一次性 helper，而是：

> **一个带有 trunk/fork/reuse 语义的专用审查会话系统。**

这也是它和 `/review` 差别进一步拉开的地方。

---

## 7. guardian 为什么还要维护 fork snapshot

guardian session manager 里还有一个很关键的点：fork snapshot。

它的作用不是普通缓存，而是：

- 把之前成功 review 后形成的上下文保存下来
- 让后续 ephemeral session 在需要时能从这个快照继承部分审查上下文

这意味着 guardian 不只是“复用一个活会话”，还支持：

> **在保持主 trunk 稳定的同时，按快照分叉出临时 reviewer。**

这是一个很像 runtime 系统而不是 prompt 工具的设计。

---

## 8. guardian 对自己的能力限制非常重

guardian 的另一个重要特征，是它虽然在“权限决定”上很重要，但它自己并不被赋予很高的执行自由。

相反，它的 session config 被明显收紧：

- `approval_policy = never`
- sandbox 为只读
- 禁用 spawn/collab
- 禁用 web search
- 只保留审查所需最小能力面

也就是说，guardian 的形态不是“超级管理员 agent”，而更像：

> **一个被严格限制的审查器。**

这也很合理：它的权力在于“给判断”，不在于“替你到处执行”。

---

## 9. approval 路由为什么会回到 parent session

guardian 还有一个特别关键的设计点：即使某个风险动作来自 delegated subagent，最终 approval authority 仍然会被收回到 parent session 处理。

也就是说：

- 子 agent 提出需要批准的动作
- parent session 决定是否走 guardian
- guardian review 在 parent control context 下发生
- 决策结果再回写给子线程

这说明 guardian 不只是“谁需要批准就谁自己审”，而是：

> **把审批权集中在 parent session 一侧。**

这样做能确保：

- 审批 authority 不会在子线程里分散
- guardian 的复用和审计也更集中
- 多 agent 协作场景下，授权控制面仍然统一

这是一个非常强的架构判断。

---

## 10. app-server / UI 看见的 guardian 不是抽象判断，而是显式事件流

guardian 不只存在于 core 内部。它在 app-server / UI 层也有清晰投影。

系统会把 guardian review 的生命周期变成可观察事件，比如：

- started
- completed
- 审查对象是什么
- 当前状态是什么
- 风险等级 / 授权信息 / rationale 是什么

在某些场景里，它还会被投影成 command execution 相关 item，让 UI 能把“某个动作为什么被拒绝/中止/超时”展示出来。

所以 guardian 不是黑箱决策器，而是：

> **有明确 app-server 协议面和 UI timeline 面的审查子系统。**

这也证明它已经从单纯内部 helper 演进成了正式基础设施。

---

## 11. analytics 为什么值得单独提一下

guardian 相关 analytics 在源码里已经有比较完整的 schema 和 reducer 支持：

- request source
- reviewed action
- decision / status
- rationale
- guardian thread id
- guardian session kind
- guardian model / reasoning effort
- timeout / token / latency 等指标

这说明设计者已经明确把 guardian 当成一个值得独立观测的系统。

不过这条线目前还有一个实际状态值得记录：

> **analytics schema 已经很完整，但 runtime 侧看起来还没有完全把这些事件都接起来。**

也就是说，观测层设计已经超前于运行时接线程度。这是这章里很有价值的一个“当前实现状态”判断。

---

## 12. `/review` 和 guardian 的最简对照

到这里，其实可以把两者压成一张很清楚的对照表。

### `/review`
- 面向用户
- 显式工作流
- 目标是产出 review 内容
- 一次性 reviewer 子线程
- 退出时形成用户可见 review output

### guardian
- 面向 approval 基础设施
- 嵌入多条审批链
- 目标是产出授权判断
- 有 trunk / fork / reuse 模式
- 结果反馈回执行链和 app-server 协议层

一句话概括：

> **review 产出评论，guardian 产出授权。**

---

## 13. 这一章最重要的判断

读完这章，应该稳定这些结论。

### 判断 1：`/review` 是用户工作流
不是 approval 基础设施。

### 判断 2：guardian 是 approval reviewer infrastructure
不是 review workflow 的别名。

### 判断 3：guardian 是 fail-closed 的
出错、超时都倾向 deny/abort，而不是乐观继续。

### 判断 4：guardian 有 trunk / ephemeral / fork snapshot 语义
说明它是可复用审查系统，而不是一次性 helper。

### 判断 5：guardian 本身被重度限权
它的权力在于判断，而不在于执行。

### 判断 6：delegated subagent 的审批权仍被收回到 parent session
approval authority 没有下放到子线程里。

### 判断 7：guardian 有正式 app-server / UI 投影面
不是内部黑箱决策器。

### 判断 8：guardian analytics 设计已成型，但接线似乎还没完全落完
这是当前实现状态的重要边界判断。

---

## 14. 下一章该接什么

review / guardian 讲完之后，接下来最值得继续拆的，不是再回头看普通执行链，而是：

> **realtime、collab 和多 agent 控制面是如何形成另一套 live runtime 的。**

也就是：

- realtime conversation manager
- handoff bridge
- AgentControl
- multi_agents_v2 handlers
- app-server 对 collab/realtime 的投影

这会自然进入下一篇：**realtime / collab / multi-agent control plane**。

---

## 相关阅读

- 想继续看 guardian analytics 这一条未接上的观测链：[[15-guardian-analytics为何还没真正接上]]
- 想继续看 realtime / collab / memories 这些更高阶系统：[[10-realtime-collab与memory迁移专题]]
- 想直接定位这条线的关键函数：[[11-关键函数索引]]、[[12-调用链索引]]

## 关键文件

- `codex-rs/core/src/tasks/review.rs`
- `codex-rs/core/src/guardian/review.rs`
- `codex-rs/core/src/guardian/review_session.rs`
- `codex-rs/core/src/codex_delegate.rs`
- `codex-rs/core/src/tools/orchestrator.rs`
- `codex-rs/core/src/tools/network_approval.rs`
- `codex-rs/core/src/mcp_tool_call.rs`
- `codex-rs/app-server/src/bespoke_event_handling.rs`
- `codex-rs/app-server-protocol/src/protocol/v2.rs`
- `codex-rs/app-server-protocol/src/protocol/thread_history.rs`
- `codex-rs/analytics/src/client.rs`
- `codex-rs/analytics/src/events.rs`
- `codex-rs/analytics/src/reducer.rs`
