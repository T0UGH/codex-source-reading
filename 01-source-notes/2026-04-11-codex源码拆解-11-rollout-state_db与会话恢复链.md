# Codex 源码拆解 11：rollout、state_db 与会话恢复链

## 这篇看什么

回答：**Codex 的会话恢复，到底是靠 rollout，还是靠 state_db，还是两者混合？**

## 先给结论

目前看到的判断已经比较清楚：

1. `rollout` 是 **会话历史真相源**
2. `state_db` 不是另一套会话历史，而是 **通过 `codex_rollout::state_db` 暴露出来的状态/索引桥**
3. `core/src/state_db_bridge.rs` 本身几乎没有业务，只是桥接层
4. 真正复杂的是 **rollout replay / reconstruction**，不是 `state_db_bridge`

一句话：

> 恢复历史靠 rollout replay；state_db 更像 rollout 周边的状态数据库桥，不是第二大脑。

## 关键代码路径

- `codex-rs/core/src/rollout.rs`
- `codex-rs/core/src/state_db_bridge.rs`
- `codex-rs/core/src/codex/rollout_reconstruction.rs`
- `codex-rs/core/src/thread_manager.rs`
- `codex-rs/core/src/codex.rs`

## 证据 1：`core/src/rollout.rs` 基本是对 `codex_rollout` 的 re-export

这里暴露：

- `RolloutRecorder`
- `ThreadItem`
- `ThreadSortKey`
- `find_thread_path_by_id_str`
- `find_thread_name_by_id`
- `read_head_for_summary`
- `SessionMeta`

以及 `Config -> RolloutConfigView` 的适配。

这说明：

- rollout 相关能力的真正实现大头不在 `core/src/rollout.rs`
- `core` 只是把 rollout crate 接进自己的 runtime 语义里

## 证据 2：`state_db_bridge.rs` 极薄

文件里基本只有：

- `pub use codex_rollout::state_db::StateDbHandle`
- `get_state_db(config)` → `codex_rollout::state_db::get_state_db(config)`

这说明一个关键事实：

**state_db 在 core 这里不是主业务层，而是导出/桥接层。**

所以如果有人把 `core/src/state_db_bridge.rs` 当作“状态恢复主链”，那会看错重点。

## 证据 3：ThreadManager 恢复线程时，先拿 rollout history

`ThreadManager` 的恢复路径：

- `resume_thread_from_rollout()`
  - `RolloutRecorder::get_rollout_history(&rollout_path)`
  - 然后调用 `resume_thread_with_history(...)`
- `resume_thread_with_history()`
  - 最终还是走 `spawn_thread(...)`

这说明：

- 恢复线程的输入是 `InitialHistory`
- 这个 `InitialHistory` 直接来自 rollout
- ThreadManager 关注的是“拿什么历史把线程重新 spawn 出来”

而不是“先查 state_db 再拼历史”。

## 证据 4：真正复杂度在 `reconstruct_history_from_rollout`

`core/src/codex/rollout_reconstruction.rs` 很有价值。

它做的不是简单顺序读取，而是：

1. **从新到旧反向扫描** rollout items
2. 找到最近仍然存活的 `replacement_history` checkpoint
3. 同时恢复：
   - `previous_turn_settings`
   - `reference_context_item`
4. 再只对“幸存后缀”正向 replay， materialize 最终 history

这里明显处理了很多复杂语义：

- compaction
- replacement history
- rollback
- aborted turn
- turn context baseline
- legacy compaction without replacement history

这说明 Codex 的恢复设计不是“读取一份快照完事”，而是：

> 用 rollout 事件流 + checkpoint 混合重建真实上下文。

## 证据 5：`replacement_history` 是恢复优化点

代码里很明确：

- 一旦发现“最新仍存活的 replacement_history checkpoint”
- 更老的 rollout items 对最终结果就不再重要

这就是一个典型的 **event log + materialized checkpoint** 设计。

它的好处：

- 不必每次从全量历史开天窗 replay
- 但仍保留足够强的可恢复语义

这比“只存快照”更稳，也比“永远全量 replay”更实用。

## 证据 6：state_db 仍然重要，但它不是历史真相源

从 `thread_manager.rs` 和 `codex.rs` 看，state_db 仍被用于：

- thread title / metadata 查询
- thread spawn descendant 查询
- dynamic tools 等周边状态读取
- session services 中的共享 `state_db` handle

所以正确理解不是“state_db 没用”，而是：

- **会话正文/历史恢复**：rollout 主导
- **结构化状态/索引/派生关系**：state_db 辅助

## 设计判断

### 1. rollout 是 durable history truth source

这个判断现在已经比上一批更稳。

### 2. state_db 更像 sidecar state index

它重要，但不是对 rollout 的平行替代。

### 3. `state_db_bridge` 这个名字很诚实

它真的只是 bridge。

如果后面要继续深挖，重点应该去：

- `codex_rollout`
- `rollout_reconstruction`
- `Codex::replace_history / reconstruct_history` 相关调用链

而不是在 core 里盯着 `state_db_bridge.rs` 发呆。

## 当前边界判断

- `rollout`：durable history + replay/checkpoint 语义
- `replacement_history`：恢复性能与复杂语义收敛点
- `state_db`：结构化状态桥 / 索引 / metadata sidecar
- `state_db_bridge`：core 对 rollout-state-db 的窄桥接层
- `ThreadManager`：拿 rollout history 重建线程

## 还值得继续拆的点

1. `codex_rollout` crate 里的 state_db 具体表结构与职责
2. rollout 与 `state/` 独立 crate 的关系边界
3. `load_thread_from_resume_source_or_send_internal` 如何把恢复态重新投影给 app-server 客户端
