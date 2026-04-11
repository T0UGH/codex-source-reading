# Codex 源码拆解 18：codex_rollout 与 state_db 内部结构

## 这篇看什么

回答：**rollout crate 自己到底长什么样？`state_db` 里面又到底存了什么？**

## 先给结论

现在可以把之前的判断再升级一层：

1. `codex-rollout` 是 **文件型 session persistence 层**
2. `codex-state` 是 **SQLite metadata / index / sidecar state 层**
3. JSONL rollout 才是会话正文
4. SQLite 主要存：
   - 线程元数据
   - 路径/标题/列表索引
   - dynamic tools
   - spawn edges
   - backfill checkpoint
5. 历史重建仍然靠 rollout item replay，不靠 SQLite 复原全文

一句话：

> rollout 负责“正文”；state_db 负责“目录、索引、派生状态”。

## 关键代码路径

- `codex-rs/rollout/src/lib.rs`
- `codex-rs/rollout/src/recorder.rs`
- `codex-rs/rollout/src/metadata.rs`
- `codex-rs/rollout/src/state_db.rs`
- `codex-rs/state/src/runtime/threads.rs`
- `codex-rs/state/src/extract.rs`
- `codex-rs/state/migrations/*.sql`

## rollout 的物理结构

### 1. 会话文件是按日期分目录的 JSONL
`recorder.rs` 里的 `precompute_log_file_info()` 会把文件写到：

- `sessions/YYYY/MM/DD/rollout-<timestamp>-<thread_id>.jsonl`

归档后则在：

- `archived_sessions/`

这说明 rollout 设计从一开始就偏：
- append-only
- 文件系统可巡检
- 可按天扫描/backfill

### 2. 还有一个 append-only 的 `session_index.jsonl`
它主要服务线程名检索：

- `append_thread_name()`
- `find_thread_name_by_id()`
- `find_thread_meta_by_name_str()`

所以除了每个 thread 自己的 rollout 文件，还额外有一个便宜的全局索引日志。

## rollout 里到底持久化什么

`RolloutRecorder::record_items()` 会按 persistence policy 过滤后写入。

当前可见的持久化对象主要有：

- `SessionMeta`
- `TurnContext`
- 选定的 `EventMsg`
- `ResponseItem`
- `Compacted`

并且受：

- `EventPersistenceMode::Limited`
- `EventPersistenceMode::Extended`

控制。

### 一个关键细节：`persist_extended_history`
app-server protocol 里已经明确说了：

- extended mode 会持久化更多 EventMsg
- 用来支持更丰富的 thread history reconstruction

这说明 rollout 不是“所有事件全存”或“只存消息”这种简单设计，而是有策略分层的。

## `replacement_history` 是 checkpoint，而不是附属字段

这是 rollout 内部最关键的点之一。

在 compaction 之后，core 会持久化：

- `CompactedItem { message, replacement_history }`

而重建历史时：

- 反向扫描 rollout
- 找到最新仍然有效的 `replacement_history`
- 把它作为 base history
- 再仅 replay 后缀

这就是典型的：

> event log + checkpoint

而不是：
- 纯 event sourcing
- 或纯 snapshot store

两边的坑都尽量绕开了。

## SQLite 里存什么

从 migration 和 runtime 代码看，`threads` 主表至少保存：

- `id`
- `rollout_path`
- `created_at`
- `updated_at`
- `source`
- `model_provider`
- latest `model`
- latest `reasoning_effort`
- `cwd`
- `cli_version`
- `title`
- `sandbox_policy`
- `approval_mode`
- `tokens_used`
- `first_user_message`
- `archived_at`
- git 相关字段
- `memory_mode`
- agent path / parent-child 相关信息

除此之外还有：

### `thread_dynamic_tools`
记录 thread-start 时的 dynamic tool specs。

### `thread_spawn_edges`
记录 agent 线程树的 parent/child 关系与状态。

### `backfill_state`
记录 rollout → SQLite backfill 的 lease / watermark / 状态。

所以现在能明确说：

**state_db 不是“把 rollout 全量结构化存进去”，而是把“适合索引/列表/关系查询的部分”结构化存进去。**

## rollout 和 state_db 是怎么同步的

不是只靠离线 backfill。

`recorder.rs` 在写 rollout item 之后，会：

- 对 metadata-affecting items 调 `state_db::apply_rollout_items(...)`
- 否则至少 `touch_thread_updated_at(...)`

这说明正常写入路径下：

- rollout 是主写入
- state_db 做增量镜像更新

而 backfill 是兜底：
- 老文件
- 丢失状态
- 路径修复
- 初次建库

## backfill 的意义

`state/src/runtime/backfill.rs` 里有：

- `try_claim_backfill(lease_seconds)`
- `checkpoint_backfill(watermark)`
- `mark_backfill_complete(...)`

这不是临时脚本级 backfill，而是正式 runtime 机制。

说明 OpenAI 很清楚：

- 文件型 rollout 和 SQLite mirror 之间天然会有重扫/修复需求
- 必须把 backfill 做成可恢复、可续跑、有 lease 的过程

## read repair 也值得注意

`rollout/src/state_db.rs` 里有：

- `find_rollout_path_by_id()`
- `read_repair_rollout_path()`
- `reconcile_rollout()`

这意味着：

如果 DB 里的路径或 metadata 过时：
- 不是直接报废
- 而是回退到文件系统再修补 DB

这套设计说明 state_db 明确是 sidecar，而不是唯一真相源。

## 当前最稳的边界判断

### `codex-rollout`
负责：
- rollout JSONL
- session meta
- thread head/summary 读取
- session index
- rollout item persistence policy
- 与 state_db 的桥接

### `codex-state`
负责：
- SQLite runtime
- schema migration
- thread metadata row
- dynamic tools / spawn edges / backfill state
- metadata extraction 应用

### core reconstruction
负责：
- 真正的 conversation history replay
- rollback / compaction / replacement_history 语义重建

## 一个更准确的图

`runtime events / response items`
→ `codex-rollout` JSONL append
→ 增量抽 metadata
→ `codex-state` SQLite mirror

而恢复时：

- list/search/path/title → 先看 SQLite
- 全文历史重建 → 回到 rollout JSONL replay

## 设计判断

### 1. 这套设计比“全 SQLite”更抗演化
因为 event/body 的 schema 演化很快，全文都塞 SQLite 反而迁移负担更大。

### 2. 这套设计比“纯文件系统扫描”更实用
因为 thread list / search / title / path lookup / dynamic tools 查询都需要结构化索引。

### 3. `state_db_bridge` 的薄，是因为真正复杂度都在 crate 下面
之前看 core 里那层桥会误以为 state_db 很轻；现在看清楚了，是复杂度下沉到了 `codex-rollout` 和 `codex-state`。

## 还值得继续观察的点

1. SQLite 当前没有看到真正的 FTS 搜索表，后续是否会扩展
2. `session_index.jsonl` 和 SQLite title 索引会不会最终统一
3. 未来是否会出现更懒加载的 rollout history 读取，而不是现在这种更偏 eager 的 replay
