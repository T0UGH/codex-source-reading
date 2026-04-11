# Codex 源码导读 Index

这不是文件堆砌索引，
而是这套材料的**总导航页**。

现在仓库里的材料已经够多了，正确用法不是从头扫到尾，
而是按目标选路径：
- 你要快速建立全局认知
- 你要只看某条主链
- 你要追到函数级边界

我下面按这三层来排。

---

## 先给结论

如果你第一次进这个仓库，先读这 3 个文件：

1. [03-boundary-judgments/2026-04-11-codex模块边界判断-v1.md](03-boundary-judgments/2026-04-11-codex模块边界判断-v1.md)
2. [STATUS.md](STATUS.md)
3. [01-source-notes/2026-04-11-codex源码拆解-01-仓库总貌与分层判断.md](01-source-notes/2026-04-11-codex源码拆解-01-仓库总貌与分层判断.md)

读完这 3 个，你应该先建立这几个总判断：
- `codex-cli/` 只是分发壳，不是主体
- `codex-rs/cli` 是统一入口层
- `core` 是 runtime 聚合中心
- `app-server` 是控制面 facade，不是 runtime 真正 owner
- 这套系统的很多一致性，不是靠一个大状态机，而是靠一批小型 reducer / projection / reconciliation helper

---

## 推荐阅读方式

### 路径 A：30 分钟快速建立全局认知
按这个顺序：

1. [03-boundary-judgments/2026-04-11-codex模块边界判断-v1.md](03-boundary-judgments/2026-04-11-codex模块边界判断-v1.md)
2. [01-source-notes/2026-04-11-codex源码拆解-01-仓库总貌与分层判断.md](01-source-notes/2026-04-11-codex源码拆解-01-仓库总貌与分层判断.md)
3. [01-source-notes/2026-04-11-codex源码拆解-05-core作为runtime聚合核心.md](01-source-notes/2026-04-11-codex源码拆解-05-core作为runtime聚合核心.md)
4. [01-source-notes/2026-04-11-codex源码拆解-10-app-server到ThreadManager的桥.md](01-source-notes/2026-04-11-codex源码拆解-10-app-server到ThreadManager的桥.md)
5. [01-source-notes/2026-04-11-codex源码拆解-18-codex_rollout与state_db内部结构.md](01-source-notes/2026-04-11-codex源码拆解-18-codex_rollout与state_db内部结构.md)
6. [01-source-notes/2026-04-11-codex源码拆解-12-exec与unified_exec的职责分工.md](01-source-notes/2026-04-11-codex源码拆解-12-exec与unified_exec的职责分工.md)
7. [01-source-notes/2026-04-11-codex源码拆解-19-plugin能力打包与skills-MCP-apps关系.md](01-source-notes/2026-04-11-codex源码拆解-19-plugin能力打包与skills-MCP-apps关系.md)
8. [01-source-notes/2026-04-11-codex源码拆解-31-backend-client与model-transport栈.md](01-source-notes/2026-04-11-codex源码拆解-31-backend-client与model-transport栈.md)

适合：
- 想先知道“这仓库到底怎么分层”
- 不想先陷进细节

### 路径 B：只看 app-server / thread / turn 主线
按这个顺序：

1. [01-source-notes/2026-04-11-codex源码拆解-09-app-server与SDK嵌入架构.md](01-source-notes/2026-04-11-codex源码拆解-09-app-server与SDK嵌入架构.md)
2. [01-source-notes/2026-04-11-codex源码拆解-10-app-server到ThreadManager的桥.md](01-source-notes/2026-04-11-codex源码拆解-10-app-server到ThreadManager的桥.md)
3. [01-source-notes/2026-04-11-codex源码拆解-15-listener与event投影链.md](01-source-notes/2026-04-11-codex源码拆解-15-listener与event投影链.md)
4. [01-source-notes/2026-04-11-codex源码拆解-16-thread_watch_manager与thread_state_manager的分工.md](01-source-notes/2026-04-11-codex源码拆解-16-thread_watch_manager与thread_state_manager的分工.md)
5. [01-source-notes/2026-04-11-codex源码拆解-17-turn-materialization与pending-request链.md](01-source-notes/2026-04-11-codex源码拆解-17-turn-materialization与pending-request链.md)
6. [01-source-notes/2026-04-11-codex源码共读-43-populate_thread_turns是turn视图装配器.md](01-source-notes/2026-04-11-codex源码共读-43-populate_thread_turns是turn视图装配器.md)
7. [01-source-notes/2026-04-11-codex源码共读-47-build_turns_from_rollout_items是turn-history回放归约器.md](01-source-notes/2026-04-11-codex源码共读-47-build_turns_from_rollout_items是turn-history回放归约器.md)
8. [01-source-notes/2026-04-11-codex源码共读-50-ThreadHistoryBuilder-handle_event是turn语义归约总分发器.md](01-source-notes/2026-04-11-codex源码共读-50-ThreadHistoryBuilder-handle_event是turn语义归约总分发器.md)
9. [01-source-notes/2026-04-11-codex源码共读-51-active_turn_snapshot是当前turn的对外投影口.md](01-source-notes/2026-04-11-codex源码共读-51-active_turn_snapshot是当前turn的对外投影口.md)
10. [01-source-notes/2026-04-11-codex源码共读-58-handle_user_message是implicit-turn收口与新输入归属器.md](01-source-notes/2026-04-11-codex源码共读-58-handle_user_message是implicit-turn收口与新输入归属器.md)

适合：
- 想搞清 listener / thread state / turn history
- 想理解 app-server 为什么能恢复、rejoin、修正状态

### 路径 C：只看 unified-exec 主线
按这个顺序：

1. [01-source-notes/2026-04-11-codex源码拆解-12-exec与unified_exec的职责分工.md](01-source-notes/2026-04-11-codex源码拆解-12-exec与unified_exec的职责分工.md)
2. [01-source-notes/2026-04-11-codex源码共读-35-UnifiedExecHandler-handle是执行请求的入口装配线.md](01-source-notes/2026-04-11-codex源码共读-35-UnifiedExecHandler-handle是执行请求的入口装配线.md)
3. [01-source-notes/2026-04-11-codex源码共读-38-UnifiedExecRuntime-run是执行请求到实际进程的最后适配层.md](01-source-notes/2026-04-11-codex源码共读-38-UnifiedExecRuntime-run是执行请求到实际进程的最后适配层.md)
4. [01-source-notes/2026-04-11-codex源码共读-42-open_session_with_exec_env是spawn边界适配器.md](01-source-notes/2026-04-11-codex源码共读-42-open_session_with_exec_env是spawn边界适配器.md)
5. [01-source-notes/2026-04-11-codex源码共读-46-from_spawned与store_process是unified-exec会话生命周期的接力点.md](01-source-notes/2026-04-11-codex源码共读-46-from_spawned与store_process是unified-exec会话生命周期的接力点.md)
6. [01-source-notes/2026-04-11-codex源码共读-49-spawn_exit_watcher与start_streaming_output是终态发射链.md](01-source-notes/2026-04-11-codex源码共读-49-spawn_exit_watcher与start_streaming_output是终态发射链.md)
7. [01-source-notes/2026-04-11-codex源码共读-52-process_chunk是输出流到transcript-delta的切片器.md](01-source-notes/2026-04-11-codex源码共读-52-process_chunk是输出流到transcript-delta的切片器.md)
8. [01-source-notes/2026-04-11-codex源码共读-53-emit_exec_end_for_unified_exec是统一exec终态封装器.md](01-source-notes/2026-04-11-codex源码共读-53-emit_exec_end_for_unified_exec是统一exec终态封装器.md)
9. [01-source-notes/2026-04-11-codex源码共读-54-emit_failed_exec_end_for_unified_exec是失败终态封装器.md](01-source-notes/2026-04-11-codex源码共读-54-emit_failed_exec_end_for_unified_exec是失败终态封装器.md)
10. [01-source-notes/2026-04-11-codex源码共读-55-resolve_aggregated_output是transcript优先的最终输出裁决器.md](01-source-notes/2026-04-11-codex源码共读-55-resolve_aggregated_output是transcript优先的最终输出裁决器.md)
11. [01-source-notes/2026-04-11-codex源码共读-56-split_valid_utf8_prefix是输出切流的UTF-8边界守门员.md](01-source-notes/2026-04-11-codex源码共读-56-split_valid_utf8_prefix是输出切流的UTF-8边界守门员.md)
12. [01-source-notes/2026-04-11-codex源码共读-57-refresh_process_state是进程存储与真实生命周期的对账器.md](01-source-notes/2026-04-11-codex源码共读-57-refresh_process_state是进程存储与真实生命周期的对账器.md)

适合：
- 想搞清 exec 和 unified-exec 的差别
- 想看 output / transcript / end-event / process-store 是怎么闭环的

---

## 主体阅读地图

下面不是“所有文件列表”，
而是按 guidebook 未来会收敛的主章节排的。

## 1. 仓库入口与总体分层

先读：
1. [01-source-notes/2026-04-11-codex源码拆解-01-仓库总貌与分层判断.md](01-source-notes/2026-04-11-codex源码拆解-01-仓库总貌与分层判断.md)
2. [01-source-notes/2026-04-11-codex源码拆解-02-为什么npm只是分发壳.md](01-source-notes/2026-04-11-codex源码拆解-02-为什么npm只是分发壳.md)
3. [01-source-notes/2026-04-11-codex源码拆解-03-Codex-CLI总入口与命令分流.md](01-source-notes/2026-04-11-codex源码拆解-03-Codex-CLI总入口与命令分流.md)
4. [00-index/01-代码路径索引-入口与分发.md](00-index/01-代码路径索引-入口与分发.md)

要解决的问题：
- 主代码到底在哪
- CLI / TUI / core / server 谁是壳，谁是主体

---

## 2. runtime layering：TUI / core / app-server / remote app-server

先读：
1. [01-source-notes/2026-04-11-codex源码拆解-04-TUI与core的边界.md](01-source-notes/2026-04-11-codex源码拆解-04-TUI与core的边界.md)
2. [01-source-notes/2026-04-11-codex源码拆解-05-core作为runtime聚合核心.md](01-source-notes/2026-04-11-codex源码拆解-05-core作为runtime聚合核心.md)
3. [01-source-notes/2026-04-11-codex源码拆解-09-app-server与SDK嵌入架构.md](01-source-notes/2026-04-11-codex源码拆解-09-app-server与SDK嵌入架构.md)
4. [01-source-notes/2026-04-11-codex源码拆解-10-app-server到ThreadManager的桥.md](01-source-notes/2026-04-11-codex源码拆解-10-app-server到ThreadManager的桥.md)
5. [01-source-notes/2026-04-11-codex源码拆解-20-TUI向app-server收敛的现状.md](01-source-notes/2026-04-11-codex源码拆解-20-TUI向app-server收敛的现状.md)
6. [01-source-notes/2026-04-11-codex源码拆解-23-remote-app-server与websocket链路.md](01-source-notes/2026-04-11-codex源码拆解-23-remote-app-server与websocket链路.md)

要解决的问题：
- runtime 真正 owner 是谁
- app-server 在系统里究竟是 facade、bridge，还是核心 runtime

---

## 3. 状态、持久化与恢复

先读：
1. [01-source-notes/2026-04-11-codex源码拆解-06-config模型与状态恢复.md](01-source-notes/2026-04-11-codex源码拆解-06-config模型与状态恢复.md)
2. [01-source-notes/2026-04-11-codex源码拆解-11-rollout-state_db与会话恢复链.md](01-source-notes/2026-04-11-codex源码拆解-11-rollout-state_db与会话恢复链.md)
3. [01-source-notes/2026-04-11-codex源码拆解-18-codex_rollout与state_db内部结构.md](01-source-notes/2026-04-11-codex源码拆解-18-codex_rollout与state_db内部结构.md)
4. [01-source-notes/2026-04-11-codex源码共读-34-reconstruct_history_from_rollout是恢复态重建器.md](01-source-notes/2026-04-11-codex源码共读-34-reconstruct_history_from_rollout是恢复态重建器.md)
5. [01-source-notes/2026-04-11-codex源码共读-47-build_turns_from_rollout_items是turn-history回放归约器.md](01-source-notes/2026-04-11-codex源码共读-47-build_turns_from_rollout_items是turn-history回放归约器.md)

要解决的问题：
- rollout、SQLite、checkpoint 各自是什么角色
- 历史恢复到底靠什么重放出来

---

## 4. app-server / listener / thread / turn 主线

先读：
1. [01-source-notes/2026-04-11-codex源码拆解-15-listener与event投影链.md](01-source-notes/2026-04-11-codex源码拆解-15-listener与event投影链.md)
2. [01-source-notes/2026-04-11-codex源码拆解-16-thread_watch_manager与thread_state_manager的分工.md](01-source-notes/2026-04-11-codex源码拆解-16-thread_watch_manager与thread_state_manager的分工.md)
3. [01-source-notes/2026-04-11-codex源码拆解-17-turn-materialization与pending-request链.md](01-source-notes/2026-04-11-codex源码拆解-17-turn-materialization与pending-request链.md)
4. [01-source-notes/2026-04-11-codex源码共读-32-ensure_listener_task_running_task是线程事件泵的安装器.md](01-source-notes/2026-04-11-codex源码共读-32-ensure_listener_task_running_task是线程事件泵的安装器.md)
5. [01-source-notes/2026-04-11-codex源码共读-33-apply_bespoke_event_handling是协议投影与副作用汇合点.md](01-source-notes/2026-04-11-codex源码共读-33-apply_bespoke_event_handling是协议投影与副作用汇合点.md)
6. [01-source-notes/2026-04-11-codex源码共读-36-handle_thread_listener_command是有序控制动作分发点.md](01-source-notes/2026-04-11-codex源码共读-36-handle_thread_listener_command是有序控制动作分发点.md)
7. [01-source-notes/2026-04-11-codex源码共读-39-handle_pending_thread_resume_request是running-thread恢复态拼装器.md](01-source-notes/2026-04-11-codex源码共读-39-handle_pending_thread_resume_request是running-thread恢复态拼装器.md)
8. [01-source-notes/2026-04-11-codex源码共读-40-resolve_pending_server_request是resolved通知的有序发射器.md](01-source-notes/2026-04-11-codex源码共读-40-resolve_pending_server_request是resolved通知的有序发射器.md)
9. [01-source-notes/2026-04-11-codex源码共读-43-populate_thread_turns是turn视图装配器.md](01-source-notes/2026-04-11-codex源码共读-43-populate_thread_turns是turn视图装配器.md)
10. [01-source-notes/2026-04-11-codex源码共读-44-set_thread_status_and_interrupt_stale_turns是活跃态校正器.md](01-source-notes/2026-04-11-codex源码共读-44-set_thread_status_and_interrupt_stale_turns是活跃态校正器.md)
11. [01-source-notes/2026-04-11-codex源码共读-45-find_and_remove_turn_summary是turn局部状态的drain边界.md](01-source-notes/2026-04-11-codex源码共读-45-find_and_remove_turn_summary是turn局部状态的drain边界.md)
12. [01-source-notes/2026-04-11-codex源码共读-48-resolve_thread_status与track_current_turn_event是一对竞态修正组合.md](01-source-notes/2026-04-11-codex源码共读-48-resolve_thread_status与track_current_turn_event是一对竞态修正组合.md)
13. [01-source-notes/2026-04-11-codex源码共读-50-ThreadHistoryBuilder-handle_event是turn语义归约总分发器.md](01-source-notes/2026-04-11-codex源码共读-50-ThreadHistoryBuilder-handle_event是turn语义归约总分发器.md)
14. [01-source-notes/2026-04-11-codex源码共读-51-active_turn_snapshot是当前turn的对外投影口.md](01-source-notes/2026-04-11-codex源码共读-51-active_turn_snapshot是当前turn的对外投影口.md)
15. [01-source-notes/2026-04-11-codex源码共读-58-handle_user_message是implicit-turn收口与新输入归属器.md](01-source-notes/2026-04-11-codex源码共读-58-handle_user_message是implicit-turn收口与新输入归属器.md)

要解决的问题：
- listener 如何把 runtime event 变成 app-server 可消费协议
- turn history 如何 live 跟踪、replay 恢复、竞态修正

---

## 5. 执行链：sandbox / exec / unified-exec / exec-server

先读：
1. [01-source-notes/2026-04-11-codex源码拆解-08-sandbox与execpolicy的分工.md](01-source-notes/2026-04-11-codex源码拆解-08-sandbox与execpolicy的分工.md)
2. [01-source-notes/2026-04-11-codex源码拆解-12-exec与unified_exec的职责分工.md](01-source-notes/2026-04-11-codex源码拆解-12-exec与unified_exec的职责分工.md)
3. [01-source-notes/2026-04-11-codex源码拆解-25-exec-server体系.md](01-source-notes/2026-04-11-codex源码拆解-25-exec-server体系.md)
4. [01-source-notes/2026-04-11-codex源码拆解-27-linux-sandbox与process-hardening实现层.md](01-source-notes/2026-04-11-codex源码拆解-27-linux-sandbox与process-hardening实现层.md)
5. [01-source-notes/2026-04-11-codex源码共读-35-UnifiedExecHandler-handle是执行请求的入口装配线.md](01-source-notes/2026-04-11-codex源码共读-35-UnifiedExecHandler-handle是执行请求的入口装配线.md)
6. [01-source-notes/2026-04-11-codex源码共读-38-UnifiedExecRuntime-run是执行请求到实际进程的最后适配层.md](01-source-notes/2026-04-11-codex源码共读-38-UnifiedExecRuntime-run是执行请求到实际进程的最后适配层.md)
7. [01-source-notes/2026-04-11-codex源码共读-42-open_session_with_exec_env是spawn边界适配器.md](01-source-notes/2026-04-11-codex源码共读-42-open_session_with_exec_env是spawn边界适配器.md)
8. [01-source-notes/2026-04-11-codex源码共读-46-from_spawned与store_process是unified-exec会话生命周期的接力点.md](01-source-notes/2026-04-11-codex源码共读-46-from_spawned与store_process是unified-exec会话生命周期的接力点.md)
9. [01-source-notes/2026-04-11-codex源码共读-49-spawn_exit_watcher与start_streaming_output是终态发射链.md](01-source-notes/2026-04-11-codex源码共读-49-spawn_exit_watcher与start_streaming_output是终态发射链.md)
10. [01-source-notes/2026-04-11-codex源码共读-52-process_chunk是输出流到transcript-delta的切片器.md](01-source-notes/2026-04-11-codex源码共读-52-process_chunk是输出流到transcript-delta的切片器.md)
11. [01-source-notes/2026-04-11-codex源码共读-53-emit_exec_end_for_unified_exec是统一exec终态封装器.md](01-source-notes/2026-04-11-codex源码共读-53-emit_exec_end_for_unified_exec是统一exec终态封装器.md)
12. [01-source-notes/2026-04-11-codex源码共读-54-emit_failed_exec_end_for_unified_exec是失败终态封装器.md](01-source-notes/2026-04-11-codex源码共读-54-emit_failed_exec_end_for_unified_exec是失败终态封装器.md)
13. [01-source-notes/2026-04-11-codex源码共读-55-resolve_aggregated_output是transcript优先的最终输出裁决器.md](01-source-notes/2026-04-11-codex源码共读-55-resolve_aggregated_output是transcript优先的最终输出裁决器.md)
14. [01-source-notes/2026-04-11-codex源码共读-56-split_valid_utf8_prefix是输出切流的UTF-8边界守门员.md](01-source-notes/2026-04-11-codex源码共读-56-split_valid_utf8_prefix是输出切流的UTF-8边界守门员.md)
15. [01-source-notes/2026-04-11-codex源码共读-57-refresh_process_state是进程存储与真实生命周期的对账器.md](01-source-notes/2026-04-11-codex源码共读-57-refresh_process_state是进程存储与真实生命周期的对账器.md)

配套草图：
- [02-call-chain-drafts/2026-04-11-codex主链草图-exec链.md](02-call-chain-drafts/2026-04-11-codex主链草图-exec链.md)

要解决的问题：
- unified-exec 相比 exec.rs 多了哪些产品语义
- output / transcript / end event / process store 怎么闭环

---

## 6. capability system：skills / plugins / MCP / apps / connectors

先读：
1. [01-source-notes/2026-04-11-codex源码拆解-07-MCP-hooks-skills的接入层.md](01-source-notes/2026-04-11-codex源码拆解-07-MCP-hooks-skills的接入层.md)
2. [01-source-notes/2026-04-11-codex源码拆解-14-skills与MCP的依赖路径.md](01-source-notes/2026-04-11-codex源码拆解-14-skills与MCP的依赖路径.md)
3. [01-source-notes/2026-04-11-codex源码拆解-19-plugin能力打包与skills-MCP-apps关系.md](01-source-notes/2026-04-11-codex源码拆解-19-plugin能力打包与skills-MCP-apps关系.md)
4. [01-source-notes/2026-04-11-codex源码拆解-21-plugin安装市场与生效链.md](01-source-notes/2026-04-11-codex源码拆解-21-plugin安装市场与生效链.md)
5. [01-source-notes/2026-04-11-codex源码拆解-26-rmcp-client与OAuth链路.md](01-source-notes/2026-04-11-codex源码拆解-26-rmcp-client与OAuth链路.md)
6. [01-source-notes/2026-04-11-codex源码拆解-30-connectors与apps子系统.md](01-source-notes/2026-04-11-codex源码拆解-30-connectors与apps子系统.md)

要解决的问题：
- plugin 是不是更高一级能力打包单元
- skills / MCP / apps / connectors 之间怎么关联

---

## 7. 协作与高级系统：review / guardian / realtime / collab / memories / agents

先读：
1. [01-source-notes/2026-04-11-codex源码拆解-22-review与guardian链路.md](01-source-notes/2026-04-11-codex源码拆解-22-review与guardian链路.md)
2. [01-source-notes/2026-04-11-codex源码拆解-29-realtime与collab子系统.md](01-source-notes/2026-04-11-codex源码拆解-29-realtime与collab子系统.md)
3. [01-source-notes/2026-04-11-codex源码拆解-24-memories-agents与external-agent-config.md](01-source-notes/2026-04-11-codex源码拆解-24-memories-agents与external-agent-config.md)

要解决的问题：
- review 和 guardian 是不是一回事
- realtime 和 collab 是不是一回事
- memories / agents 到底处在启动期还是 session 期

---

## 8. 可观测性与传输底层

先读：
1. [01-source-notes/2026-04-11-codex源码拆解-28-analytics与telemetry归约链.md](01-source-notes/2026-04-11-codex源码拆解-28-analytics与telemetry归约链.md)
2. [01-source-notes/2026-04-11-codex源码拆解-31-backend-client与model-transport栈.md](01-source-notes/2026-04-11-codex源码拆解-31-backend-client与model-transport栈.md)

要解决的问题：
- analytics 为什么说是 reducer-centered
- model transport 与 backend-client 为什么不是一条线

---

## 辅助材料

- [00-index/00-Codex源码拆解总索引.md](00-index/00-Codex源码拆解总索引.md)
- [00-index/01-代码路径索引-入口与分发.md](00-index/01-代码路径索引-入口与分发.md)
- [02-call-chain-drafts/2026-04-11-codex主链草图-interactive-TUI链.md](02-call-chain-drafts/2026-04-11-codex主链草图-interactive-TUI链.md)
- [02-call-chain-drafts/2026-04-11-codex主链草图-exec链.md](02-call-chain-drafts/2026-04-11-codex主链草图-exec链.md)
- [STATUS.md](STATUS.md)

---

## 现在的建议

这份仓库接下来不要再继续横向铺更多 source notes 了。

更合理的下一步是：
1. 以这份 index 为入口
2. 把现有材料重组为一份连续可读的 guidebook
3. 把函数级共读降为附录/证据层，而不是主阅读层

一句话说：

> **Phase 1 的“拆散、验证、补证据”已经差不多了，下一阶段应该是“重组、讲顺、形成读者友好的主线”。**
