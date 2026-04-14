---
title: Codex guidebook 11｜关键函数索引
date: 2026-04-12
status: draft
purpose: appendix
---

# 11｜关键函数索引

这不是按字母排序的 API 列表，
而是这套导读里最值得反复回看的 **runtime pivot 索引**。

使用方式：
- 如果你已经读过正文，想追源码证据，从这里按主题下钻
- 如果你只想快速定位某条主线的“关键函数”，从这里反查最值的入口

---

## 1. app-server / listener / thread / turn 主线

### `ensure_listener_task_running_task(...)`
角色：线程事件泵的安装器。
为什么重要：决定一个 thread 的 listener 是怎么被挂起来的，是 app-server 主链的起点之一。
推荐搭配：
- `03-app-server与thread-turn主线`
- `2026-04-11-codex源码共读-32-...`

### `apply_bespoke_event_handling(...)`
角色：协议投影与副作用汇合点。
为什么重要：把 core `EventMsg` 投影成 app-server 可消费的协议通知，是 facade 层最关键的翻译器。
推荐搭配：
- `03-app-server与thread-turn主线`
- `2026-04-11-codex源码共读-33-...`

### `handle_thread_listener_command(...)`
角色：有序控制动作分发点。
为什么重要：把 resume / resolve request 之类不能乱序的控制动作串回 listener 上下文。
推荐搭配：
- `03-app-server与thread-turn主线`
- `2026-04-11-codex源码共读-36-...`

### `handle_pending_thread_resume_request(...)`
角色：running-thread 恢复态拼装器。
为什么重要：解释为什么 running thread resume 不是“外面拼响应”，而是 listener context 串行化。
推荐搭配：
- `03-app-server与thread-turn主线`
- `2026-04-11-codex源码共读-39-...`

### `resolve_pending_server_request(...)`
角色：resolved 通知的有序发射器。
为什么重要：pending request replay / resolve 的顺序正确性靠它维持。
推荐搭配：
- `03-app-server与thread-turn主线`
- `2026-04-11-codex源码共读-40-...`

### `populate_thread_turns(...)`
角色：turn 视图装配器。
为什么重要：把 rollout 历史和 live active turn 拼成对外可见 history。
推荐搭配：
- `03-app-server与thread-turn主线`
- `2026-04-11-codex源码共读-43-...`

### `set_thread_status_and_interrupt_stale_turns(...)`
角色：活跃态校正器。
为什么重要：修正 stale in-progress turn 与真实 runtime 状态的偏差。
推荐搭配：
- `03-app-server与thread-turn主线`
- `2026-04-11-codex源码共读-44-...`

### `resolve_thread_status(...)`
角色：竞态修正器。
为什么重要：watch 状态滞后时，优先反映真实 in-progress turn。
推荐搭配：
- `03-app-server与thread-turn主线`
- `2026-04-11-codex源码共读-48-...`

---

## 2. turn-history 语义层

### `build_turns_from_rollout_items(...)`
角色：turn-history 回放归约入口。
为什么重要：从 persisted rollout 进入 turn 语义世界的第一道门。
推荐搭配：
- `04-turn-history语义层`
- `2026-04-11-codex源码共读-47-...`

### `ThreadHistoryBuilder::handle_event(...)`
角色：turn 语义归约总分发器。
为什么重要：turn-history 语义权威的核心函数。
推荐搭配：
- `04-turn-history语义层`
- `2026-04-11-codex源码共读-50-...`

### `active_turn_snapshot(...)`
角色：当前/最近 turn 的对外投影口。
为什么重要：running-thread resume 与 live turn overlay 的桥点。
推荐搭配：
- `04-turn-history语义层`
- `2026-04-11-codex源码共读-51-...`

### `handle_user_message(...)`
角色：implicit-turn 收口与新输入归属器。
为什么重要：旧流兼容下最重要的 turn 边界函数之一。
推荐搭配：
- `04-turn-history语义层`
- `2026-04-11-codex源码共读-58-...`

### `Session::reconstruct_history_from_rollout(...)`
角色：恢复态重建器。
为什么重要：解释 rollback / compaction / surviving history 的 persisted semantics。
推荐搭配：
- `02-状态持久化与恢复`
- `04-turn-history语义层`
- `2026-04-11-codex源码共读-34-...`

---

## 3. unified-exec 执行子系统

### `UnifiedExecHandler::handle(...)`
角色：执行请求的入口装配线。
为什么重要：普通 tool call 进入 unified-exec 子系统的第一道门。
推荐搭配：
- `05-unified-exec执行子系统`
- `2026-04-11-codex源码共读-35-...`

### `UnifiedExecRuntime::run(...)`
角色：执行请求到实际进程的最后适配层。
为什么重要：approval / policy / run path 的交汇点。
推荐搭配：
- `05-unified-exec执行子系统`
- `2026-04-11-codex源码共读-38-...`

### `open_session_with_exec_env(...)`
角色：spawn 边界适配器。
为什么重要：把执行请求接到本地/远端真实执行环境上。
推荐搭配：
- `05-unified-exec执行子系统`
- `2026-04-11-codex源码共读-42-...`

### `UnifiedExecProcess::from_spawned(...)` / `store_process(...)`
角色：会话生命周期接力点。
为什么重要：证明 unified-exec 是 sessionful subsystem，不是一次性调用。
推荐搭配：
- `05-unified-exec执行子系统`
- `2026-04-11-codex源码共读-46-...`

### `spawn_exit_watcher(...)` / `start_streaming_output(...)`
角色：终态发射链。
为什么重要：输出流和结束态是怎么挂起来的。
推荐搭配：
- `05-unified-exec执行子系统`
- `2026-04-11-codex源码共读-49-...`

### `process_chunk(...)`
角色：输出流到 transcript-delta 的切片器。
为什么重要：输出正确性不靠大对象，而靠这种小边界函数。
推荐搭配：
- `05-unified-exec执行子系统`
- `2026-04-11-codex源码共读-52-...`

### `emit_exec_end_for_unified_exec(...)` / `emit_failed_exec_end_for_unified_exec(...)`
角色：统一终态封装器。
为什么重要：success/failure 最终如何被协议化。
推荐搭配：
- `05-unified-exec执行子系统`
- `2026-04-11-codex源码共读-53-54-...`

### `resolve_aggregated_output(...)`
角色：transcript 优先的最终输出裁决器。
为什么重要：证明 transcript 比 process store 更接近最终语义真相。
推荐搭配：
- `05-unified-exec执行子系统`
- `2026-04-11-codex源码共读-55-...`

### `split_valid_utf8_prefix(...)`
角色：UTF-8 边界守门员。
为什么重要：streaming output 为什么没有在字符边界上崩坏。
推荐搭配：
- `05-unified-exec执行子系统`
- `2026-04-11-codex源码共读-56-...`

### `refresh_process_state(...)`
角色：进程存储与真实生命周期的对账器。
为什么重要：说明 process store 不是绝对真相源。
推荐搭配：
- `05-unified-exec执行子系统`
- `2026-04-11-codex源码共读-57-...`

---

## 4. 模型请求 / transport 专题

### `ModelClient::new_session()`
角色：turn-aware 模型请求 session 创建点。
为什么重要：把 session 级 websocket/fallback 语义接到 turn 上。
推荐搭配：
- `07-model-client与provider请求主链`

### `build_responses_request(...)`
角色：Responses 请求体装配器。
为什么重要：模型请求先组语义，再进 transport。
推荐搭配：
- `07-model-client与provider请求主链`

### `build_responses_options(...)`
角色：turn/session 头信息装配器。
为什么重要：conversation id、turn state、subagent metadata 都在这里挂到请求上。
推荐搭配：
- `07-model-client与provider请求主链`

### `stream_responses_websocket(...)`
角色：WebSocket 主路径。
为什么重要：WebSocket 在 Codex 里承载的是 turn 连续性，而不只是更快 transport。
推荐搭配：
- `07-model-client与provider请求主链`

### `EndpointSession::{execute_with, stream_with}`
角色：`codex-api` 的公共 endpoint seam。
为什么重要：解释为什么 `codex-api` 不是散乱 endpoint 集合。
推荐搭配：
- `08-codex-client-codex-api与backend-client分层`

---

## 5. review / guardian 专题

### `ReviewTask::run()`
角色：`/review` 工作流入口。
为什么重要：用户工作流与 approval infrastructure 的边界，从这里开始分开。
推荐搭配：
- `09-review工作流与guardian审查基础设施`

### `run_guardian_review(...)`
角色：guardian 审查主链。
为什么重要：fail-closed、评估结果与 approval decision 的核心入口。
推荐搭配：
- `09-review工作流与guardian审查基础设施`

### `run_review_on_session(...)`
角色：guardian 审查在 trunk/ephemeral session 上的真正执行体。
为什么重要：reuse、delta prompt、prior context 都从这里落地。
推荐搭配：
- `09-review工作流与guardian审查基础设施`

### `AgentControl::spawn_agent_internal(...)`
角色：rooted thread-tree control plane 的 spawn 主链。
为什么重要：review/guardian/collab 很多子线程语义最终都要落回控制面。
推荐搭配：
- `09-review工作流与guardian审查基础设施`
- `10-realtime-collab与memory迁移专题`

---

## 6. realtime / collab / memories 专题

### `RealtimeConversationManager::start(...)`
角色：实时会话启动入口。
为什么重要：realtime 是独立 conversation control plane，不是普通 transport 附件。
推荐搭配：
- `10-realtime-collab与memory迁移专题`

### `RealtimeResponseCreateQueue`
角色：实时 response 调度器。
为什么重要：backend active-response 约束被本地小对象吸收。
推荐搭配：
- `10-realtime-collab与memory迁移专题`

### `Session::route_realtime_text_input(...)`
角色：handoff 回接主 turn 引擎的桥。
为什么重要：realtime 没有形成平行世界，而是能回到普通 session 主链。
推荐搭配：
- `10-realtime-collab与memory迁移专题`

### `AgentControl`
角色：multi-agent rooted thread-tree control plane。
为什么重要：multi_agents_v2 handlers 只是 facade，真正的 owner 在这里。
推荐搭配：
- `10-realtime-collab与memory迁移专题`

### `start_memories_startup_task(...)`
角色：memories 启动管线统一入口。
为什么重要：memories 是 startup pipeline，不是 turn-time helper。
推荐搭配：
- `10-realtime-collab与memory迁移专题`

### `claim_stage1_jobs_for_startup(...)` / `try_claim_global_phase2_job(...)`
角色：memories durable workflow 协调关键点。
为什么重要：phase1 extract 与 phase2 global consolidation 的 durable coordination 都在这里落地。
推荐搭配：
- `10-realtime-collab与memory迁移专题`

### `ExternalAgentConfigService::{detect, import}`
角色：migration/compatibility 服务入口。
为什么重要：说明 external-agent-config 是迁移层，而不是主 runtime spine。
推荐搭配：
- `10-realtime-collab与memory迁移专题`

---

## 怎么用这份索引

### 如果你想继续深挖主骨架
优先回看：
- listener / turn-history / unified-exec 这三组函数

### 如果你想补 transport / backend 边界
优先回看：
- `ModelClient` + `EndpointSession` + websocket path

### 如果你想补高级系统
优先回看：
- guardian 审查主链
- realtime manager
- `AgentControl`
- memories phase1/phase2 claim 点

---

## 相关阅读

- 主干正文：[[01-系统总图与分层]]、[[03-app-server与thread-turn主线]]、[[05-unified-exec执行子系统]]
- 专题正文：[[09-review工作流与guardian审查基础设施]]、[[10-realtime-collab与memory迁移专题]]
- 附录配套：[[12-调用链索引]]、[[13-open-questions与后续深挖方向]]
