# Codex 源码拆解 Index

这不是流水账索引，而是**阅读导航页**。

如果你第一次进这个仓库，建议按下面顺序读。

---

## 0. 先看结论页

- [03-boundary-judgments/2026-04-11-codex模块边界判断-v1.md](03-boundary-judgments/2026-04-11-codex模块边界判断-v1.md)
- [STATUS.md](STATUS.md)

这两篇先帮你建立：
- 当前最稳的边界判断
- 当前已经拆到哪
- 还有哪些开放问题

---

## 1. 仓库总图：先搞清主体在哪

建议顺序：

1. [01-source-notes/2026-04-11-codex源码拆解-01-仓库总貌与分层判断.md](01-source-notes/2026-04-11-codex源码拆解-01-仓库总貌与分层判断.md)
2. [01-source-notes/2026-04-11-codex源码拆解-02-为什么npm只是分发壳.md](01-source-notes/2026-04-11-codex源码拆解-02-为什么npm只是分发壳.md)
3. [01-source-notes/2026-04-11-codex源码拆解-03-Codex-CLI总入口与命令分流.md](01-source-notes/2026-04-11-codex源码拆解-03-Codex-CLI总入口与命令分流.md)
4. [00-index/01-代码路径索引-入口与分发.md](00-index/01-代码路径索引-入口与分发.md)

读完这组，你应该明确：
- `codex-cli/` 不是主体
- `codex-rs/cli` 是统一入口
- `core` 才是 runtime 核心

---

## 2. 核心架构：TUI / core / app-server / mcp-server

建议顺序：

1. [01-source-notes/2026-04-11-codex源码拆解-04-TUI与core的边界.md](01-source-notes/2026-04-11-codex源码拆解-04-TUI与core的边界.md)
2. [01-source-notes/2026-04-11-codex源码拆解-05-core作为runtime聚合核心.md](01-source-notes/2026-04-11-codex源码拆解-05-core作为runtime聚合核心.md)
3. [01-source-notes/2026-04-11-codex源码拆解-09-app-server与SDK嵌入架构.md](01-source-notes/2026-04-11-codex源码拆解-09-app-server与SDK嵌入架构.md)
4. [01-source-notes/2026-04-11-codex源码拆解-10-app-server到ThreadManager的桥.md](01-source-notes/2026-04-11-codex源码拆解-10-app-server到ThreadManager的桥.md)
5. [01-source-notes/2026-04-11-codex源码拆解-13-mcp-server与app-server的产品边界.md](01-source-notes/2026-04-11-codex源码拆解-13-mcp-server与app-server的产品边界.md)
6. [01-source-notes/2026-04-11-codex源码拆解-20-TUI向app-server收敛的现状.md](01-source-notes/2026-04-11-codex源码拆解-20-TUI向app-server收敛的现状.md)
7. [01-source-notes/2026-04-11-codex源码拆解-23-remote-app-server与websocket链路.md](01-source-notes/2026-04-11-codex源码拆解-23-remote-app-server与websocket链路.md)

读完这组，你应该明确：
- `ThreadManager` 才是 runtime 入口
- `app-server` 是控制面 facade
- `mcp-server` 是 toolified 暴露面
- TUI 主链已经大幅 app-server 化

---

## 3. 状态与持久化：config / rollout / state_db / 恢复

建议顺序：

1. [01-source-notes/2026-04-11-codex源码拆解-06-config模型与状态恢复.md](01-source-notes/2026-04-11-codex源码拆解-06-config模型与状态恢复.md)
2. [01-source-notes/2026-04-11-codex源码拆解-11-rollout-state_db与会话恢复链.md](01-source-notes/2026-04-11-codex源码拆解-11-rollout-state_db与会话恢复链.md)
3. [01-source-notes/2026-04-11-codex源码拆解-18-codex_rollout与state_db内部结构.md](01-source-notes/2026-04-11-codex源码拆解-18-codex_rollout与state_db内部结构.md)

读完这组，你应该明确：
- rollout JSONL 是正文真相源
- SQLite 是 metadata/index/sidecar
- 历史恢复靠 replay + checkpoint，不靠纯 DB

---

## 4. 事件与线程状态：listener / turn / pending request

建议顺序：

1. [01-source-notes/2026-04-11-codex源码拆解-15-listener与event投影链.md](01-source-notes/2026-04-11-codex源码拆解-15-listener与event投影链.md)
2. [01-source-notes/2026-04-11-codex源码拆解-16-thread_watch_manager与thread_state_manager的分工.md](01-source-notes/2026-04-11-codex源码拆解-16-thread_watch_manager与thread_state_manager的分工.md)
3. [01-source-notes/2026-04-11-codex源码拆解-17-turn-materialization与pending-request链.md](01-source-notes/2026-04-11-codex源码拆解-17-turn-materialization与pending-request链.md)

读完这组，你应该明确：
- listener 是事件泵
- `bespoke_event_handling` 是协议投影层
- turn 状态是 event 累积 + finalization
- pending request 有 replay / abort / resolve 语义

---

## 5. 执行链：sandbox / exec / unified_exec / exec-server

建议顺序：

1. [01-source-notes/2026-04-11-codex源码拆解-08-sandbox与execpolicy的分工.md](01-source-notes/2026-04-11-codex源码拆解-08-sandbox与execpolicy的分工.md)
2. [01-source-notes/2026-04-11-codex源码拆解-12-exec与unified_exec的职责分工.md](01-source-notes/2026-04-11-codex源码拆解-12-exec与unified_exec的职责分工.md)
3. [01-source-notes/2026-04-11-codex源码拆解-25-exec-server体系.md](01-source-notes/2026-04-11-codex源码拆解-25-exec-server体系.md)
4. [01-source-notes/2026-04-11-codex源码拆解-27-linux-sandbox与process-hardening实现层.md](01-source-notes/2026-04-11-codex源码拆解-27-linux-sandbox与process-hardening实现层.md)

配套草图：
- [02-call-chain-drafts/2026-04-11-codex主链草图-exec链.md](02-call-chain-drafts/2026-04-11-codex主链草图-exec链.md)

读完这组，你应该明确：
- `exec.rs` 是底座
- `unified_exec` 是会话化执行产品
- `exec-server` 是 process/filesystem RPC backend
- Linux sandbox 当前是 bwrap-first

---

## 6. 能力接入：skills / plugins / MCP / apps / connectors

建议顺序：

1. [01-source-notes/2026-04-11-codex源码拆解-07-MCP-hooks-skills的接入层.md](01-source-notes/2026-04-11-codex源码拆解-07-MCP-hooks-skills的接入层.md)
2. [01-source-notes/2026-04-11-codex源码拆解-14-skills与MCP的依赖路径.md](01-source-notes/2026-04-11-codex源码拆解-14-skills与MCP的依赖路径.md)
3. [01-source-notes/2026-04-11-codex源码拆解-19-plugin能力打包与skills-MCP-apps关系.md](01-source-notes/2026-04-11-codex源码拆解-19-plugin能力打包与skills-MCP-apps关系.md)
4. [01-source-notes/2026-04-11-codex源码拆解-21-plugin安装市场与生效链.md](01-source-notes/2026-04-11-codex源码拆解-21-plugin安装市场与生效链.md)
5. [01-source-notes/2026-04-11-codex源码拆解-26-rmcp-client与OAuth链路.md](01-source-notes/2026-04-11-codex源码拆解-26-rmcp-client与OAuth链路.md)
6. [01-source-notes/2026-04-11-codex源码拆解-30-connectors与apps子系统.md](01-source-notes/2026-04-11-codex源码拆解-30-connectors与apps子系统.md)

读完这组，你应该明确：
- plugin 是更上游的 capability packaging unit
- MCP client/server 是两条不同链
- apps/connectors 是目录态 + 运行态 + plugin 声明态的合并结果

---

## 7. 协作与高级系统：review / guardian / realtime / collab / memories / agents

建议顺序：

1. [01-source-notes/2026-04-11-codex源码拆解-22-review与guardian链路.md](01-source-notes/2026-04-11-codex源码拆解-22-review与guardian链路.md)
2. [01-source-notes/2026-04-11-codex源码拆解-29-realtime与collab子系统.md](01-source-notes/2026-04-11-codex源码拆解-29-realtime与collab子系统.md)
3. [01-source-notes/2026-04-11-codex源码拆解-24-memories-agents与external-agent-config.md](01-source-notes/2026-04-11-codex源码拆解-24-memories-agents与external-agent-config.md)

读完这组，你应该明确：
- `/review` 和 guardian 是两套系统
- realtime 和 collab 也不是一回事
- memories 是启动期 pipeline
- agents 是 session-scoped control plane

---

## 8. 可观测性与传输底层

建议顺序：

1. [01-source-notes/2026-04-11-codex源码拆解-28-analytics与telemetry归约链.md](01-source-notes/2026-04-11-codex源码拆解-28-analytics与telemetry归约链.md)
2. [01-source-notes/2026-04-11-codex源码拆解-31-backend-client与model-transport栈.md](01-source-notes/2026-04-11-codex源码拆解-31-backend-client与model-transport栈.md)

读完这组，你应该明确：
- analytics 是 reducer-centered
- model transport 是 `codex-client -> codex-api -> ModelClient`
- `backend-client` 是另一条后端 API client 线

---

## 9. 草图与辅助材料

- [02-call-chain-drafts/2026-04-11-codex主链草图-interactive-TUI链.md](02-call-chain-drafts/2026-04-11-codex主链草图-interactive-TUI链.md)
- [02-call-chain-drafts/2026-04-11-codex主链草图-exec链.md](02-call-chain-drafts/2026-04-11-codex主链草图-exec链.md)
- [00-index/00-Codex源码拆解总索引.md](00-index/00-Codex源码拆解总索引.md)

---

## 推荐阅读路径

### 路径 A：只想快速建立全局认知
按这个顺序：

1. 边界判断 v1
2. 仓库总貌
3. core 作为 runtime 核心
4. app-server 到 ThreadManager 的桥
5. rollout/state_db 内部结构
6. plugin 能力打包
7. review 与 guardian
8. model transport 栈

### 路径 B：只关心执行链
按这个顺序：

1. sandbox 与 execpolicy
2. exec 与 unified_exec
3. exec-server
4. linux-sandbox
5. remote app-server 与 websocket
6. rmcp-client 与 OAuth

### 路径 C：只关心 agent/runtime 协作
按这个顺序：

1. listener 与 event 投影链
2. thread state/watch 分工
3. turn materialization
4. review 与 guardian
5. realtime 与 collab
6. memories / agents / external-agent-config

---

## 现在最值得做的下一步

不是继续横向扩主题，而是：

1. 把这 31 篇架构/实现稿按章节重组
2. 把新的函数级共读稿继续补成一组稳定系列
3. 补关键 call-chain 图
4. 最后再写成一份真正连续可读的 Codex 架构指南

---

## 附：函数级源码共读（更细颗粒度）

如果你觉得上面的架构稿还不够细，先从这一组开始：

1. [01-source-notes/2026-04-11-codex源码共读-32-ensure_listener_task_running_task是线程事件泵的安装器.md](01-source-notes/2026-04-11-codex源码共读-32-ensure_listener_task_running_task是线程事件泵的安装器.md)
2. [01-source-notes/2026-04-11-codex源码共读-33-apply_bespoke_event_handling是协议投影与副作用汇合点.md](01-source-notes/2026-04-11-codex源码共读-33-apply_bespoke_event_handling是协议投影与副作用汇合点.md)
3. [01-source-notes/2026-04-11-codex源码共读-34-reconstruct_history_from_rollout是恢复态重建器.md](01-source-notes/2026-04-11-codex源码共读-34-reconstruct_history_from_rollout是恢复态重建器.md)
4. [01-source-notes/2026-04-11-codex源码共读-35-UnifiedExecHandler-handle是执行请求的入口装配线.md](01-source-notes/2026-04-11-codex源码共读-35-UnifiedExecHandler-handle是执行请求的入口装配线.md)
5. [01-source-notes/2026-04-11-codex源码共读-36-handle_thread_listener_command是有序控制动作分发点.md](01-source-notes/2026-04-11-codex源码共读-36-handle_thread_listener_command是有序控制动作分发点.md)
6. [01-source-notes/2026-04-11-codex源码共读-37-handle_turn_complete是turn终态封口器.md](01-source-notes/2026-04-11-codex源码共读-37-handle_turn_complete是turn终态封口器.md)
7. [01-source-notes/2026-04-11-codex源码共读-38-UnifiedExecRuntime-run是执行请求到实际进程的最后适配层.md](01-source-notes/2026-04-11-codex源码共读-38-UnifiedExecRuntime-run是执行请求到实际进程的最后适配层.md)
8. [01-source-notes/2026-04-11-codex源码共读-39-handle_pending_thread_resume_request是running-thread恢复态拼装器.md](01-source-notes/2026-04-11-codex源码共读-39-handle_pending_thread_resume_request是running-thread恢复态拼装器.md)
9. [01-source-notes/2026-04-11-codex源码共读-40-resolve_pending_server_request是resolved通知的有序发射器.md](01-source-notes/2026-04-11-codex源码共读-40-resolve_pending_server_request是resolved通知的有序发射器.md)
10. [01-source-notes/2026-04-11-codex源码共读-41-handle_error是turn失败语义的写入点.md](01-source-notes/2026-04-11-codex源码共读-41-handle_error是turn失败语义的写入点.md)
11. [01-source-notes/2026-04-11-codex源码共读-42-open_session_with_exec_env是spawn边界适配器.md](01-source-notes/2026-04-11-codex源码共读-42-open_session_with_exec_env是spawn边界适配器.md)
12. [01-source-notes/2026-04-11-codex源码共读-43-populate_thread_turns是turn视图装配器.md](01-source-notes/2026-04-11-codex源码共读-43-populate_thread_turns是turn视图装配器.md)
13. [01-source-notes/2026-04-11-codex源码共读-44-set_thread_status_and_interrupt_stale_turns是活跃态校正器.md](01-source-notes/2026-04-11-codex源码共读-44-set_thread_status_and_interrupt_stale_turns是活跃态校正器.md)
14. [01-source-notes/2026-04-11-codex源码共读-45-find_and_remove_turn_summary是turn局部状态的drain边界.md](01-source-notes/2026-04-11-codex源码共读-45-find_and_remove_turn_summary是turn局部状态的drain边界.md)
15. [01-source-notes/2026-04-11-codex源码共读-46-from_spawned与store_process是unified-exec会话生命周期的接力点.md](01-source-notes/2026-04-11-codex源码共读-46-from_spawned与store_process是unified-exec会话生命周期的接力点.md)
16. [01-source-notes/2026-04-11-codex源码共读-47-build_turns_from_rollout_items是turn-history回放归约器.md](01-source-notes/2026-04-11-codex源码共读-47-build_turns_from_rollout_items是turn-history回放归约器.md)
17. [01-source-notes/2026-04-11-codex源码共读-48-resolve_thread_status与track_current_turn_event是一对竞态修正组合.md](01-source-notes/2026-04-11-codex源码共读-48-resolve_thread_status与track_current_turn_event是一对竞态修正组合.md)
18. [01-source-notes/2026-04-11-codex源码共读-49-spawn_exit_watcher与start_streaming_output是终态发射链.md](01-source-notes/2026-04-11-codex源码共读-49-spawn_exit_watcher与start_streaming_output是终态发射链.md)
19. [01-source-notes/2026-04-11-codex源码共读-50-ThreadHistoryBuilder-handle_event是turn语义归约总分发器.md](01-source-notes/2026-04-11-codex源码共读-50-ThreadHistoryBuilder-handle_event是turn语义归约总分发器.md)
20. [01-source-notes/2026-04-11-codex源码共读-51-active_turn_snapshot是当前turn的对外投影口.md](01-source-notes/2026-04-11-codex源码共读-51-active_turn_snapshot是当前turn的对外投影口.md)
21. [01-source-notes/2026-04-11-codex源码共读-52-process_chunk是输出流到transcript-delta的切片器.md](01-source-notes/2026-04-11-codex源码共读-52-process_chunk是输出流到transcript-delta的切片器.md)
22. [01-source-notes/2026-04-11-codex源码共读-53-emit_exec_end_for_unified_exec是统一exec终态封装器.md](01-source-notes/2026-04-11-codex源码共读-53-emit_exec_end_for_unified_exec是统一exec终态封装器.md)

这组的目标不是再讲模块边界，而是把单个关键函数在主链里的角色、状态迁移和设计意图拆清楚。
