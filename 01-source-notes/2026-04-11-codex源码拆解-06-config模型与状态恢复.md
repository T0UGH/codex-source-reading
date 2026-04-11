# Codex 源码拆解 06：config 模型与状态恢复

## 这篇看什么

回答：**Codex 怎么加载配置、怎么分层覆盖、怎么恢复会话状态？`config` 和 `state` 为什么要拆成两个 crate？**

## 先给结论

Codex 在这块明显做了两层拆分：

- `codex-config`：负责 **配置 schema / layer / merge / provenance / requirements 表达**
- `codex-state`：负责 **SQLite-backed rollout metadata mirror**，也就是把 rollout/session 元数据镜像进本地状态库

而真正把两者接进 runtime 的，是 `codex-core`。

这意味着：
- config 不是简单 TOML parse
- state 不是完整 session snapshot
- session 恢复更像 **rollout reconstruction + metadata sidecar**

## 关键代码路径

- `codex-rs/config/src/lib.rs`
- `codex-rs/config/src/config_toml.rs`
- `codex-rs/config/src/state.rs`
- `codex-rs/core/src/config_loader/mod.rs`
- `codex-rs/core/src/config/mod.rs`
- `codex-rs/state/src/lib.rs`
- `codex-rs/state/src/runtime.rs`
- `codex-rs/state/src/model/thread_metadata.rs`
- `codex-rs/state/src/extract.rs`
- `codex-rs/state/src/runtime/threads.rs`
- `codex-rs/core/src/codex.rs`
- `codex-rs/core/src/codex/rollout_reconstruction.rs`
- `codex-rs/core/src/thread_manager.rs`

## config 模型判断

### 1. `codex-config` 负责 schema 与层叠合并
它暴露的不是单一 `Config`，而是：
- raw TOML types
- layer stack
- layer provenance
- requirements model
- merge/edit utilities

特别是 `config/src/state.rs` 里的：
- `ConfigLayerEntry`
- `ConfigLayerStack`
- `effective_config()`
- `origins()`

这说明 Codex 明确在建模：
> 配置从哪来、哪层覆盖哪层、每个字段最终来源是谁。

### 2. 真正 runtime 生效的 `Config` 在 `core` 里生成
`core/src/config_loader/mod.rs` 负责发现并加载 layer。
`core/src/config/mod.rs` 的 `ConfigBuilder::build()` 再把 merged TOML 变成 runtime `Config`。

这说明：
- `codex-config` 更偏“配置语言与结构”
- `core` 更偏“运行时最终使用的配置对象”

## state 模型判断

### 1. `codex-state` 是 SQLite metadata mirror，不是完整 session dump
`state/src/lib.rs` 直接写明：
- 它从 rollout JSONL 抽 metadata
- 再镜像到本地 SQLite DB
- backfill orchestration / rollout scanning 在 `core`

这点很关键。

所以不能把 `state` 理解成：
> “把整个运行时快照完整存盘”

它更像：
> “把重要的会话/线程元数据抽出来，做成本地可查询状态库”

### 2. `StateRuntime` 管两套 DB
`state/src/runtime.rs` 显示：
- `state_5.sqlite`
- `logs_2.sqlite`

说明它区分了：
- 运行状态/元数据
- 日志数据库

### 3. `ThreadMetadata` 是很重要的中心对象
这里保存了：
- thread id
- rollout path
- created/updated
- source
- provider/model/reasoning
- cwd
- title
- sandbox/approval mode
- tokens used
- git 信息

这基本就是 Codex 对“一个 thread 的可索引摘要”的正式建模。

## 会话恢复判断

### 1. 恢复不是“直接读取一份完整 snapshot”
当前更像：
- rollout 文件提供历史事实
- state DB 提供 metadata sidecar
- `core` 在恢复时做 reconstruction

### 2. resume/fork 走的是 rollout reconstruction
关键路径：
- `ThreadManager::resume_thread_from_rollout(...)`
- `Session::record_initial_history(...)`
- `core/src/codex/rollout_reconstruction.rs`

这说明恢复逻辑是：
1. 找到 rollout history
2. 解析 surviving history / previous turn settings / rollback effects
3. 重建 `ContextManager`
4. 恢复 thread 继续运行

### 3. session state 有一部分是内存态，不会被原样持久化
`core/src/state/session.rs` 的 `SessionState` 里有很多运行时字段：
- history
- granted permissions
- latest rate limits
- previous turn settings
- connector selection

这说明恢复并不是 100% snapshot replay，
而是：
- 一部分来自 durable rollout
- 一部分来自 metadata sidecar
- 一部分是 best-effort rebuild

## 设计判断

### 1. config / state 拆 crate 是对的
- config 解决“怎么表达与合并配置”
- state 解决“怎么索引和查询持久化元数据”
- core 解决“怎么把这些东西真正用于 runtime”

### 2. rollout 是第一真相源，SQLite 是查询友好镜像层
这比“直接全塞 SQLite”更接近 agent runtime 的真实需求。

### 3. 会话恢复是 reconstruction，不是反序列化
这点很重要，因为它会直接影响你后面对 Codex “可恢复性”的理解方式。

## 当前边界判断

- `codex-config`：配置 schema/layer/provenance
- `codex-state`：SQLite metadata/log mirror
- `core`：config/state 真正接入运行时的地方
- rollout：durable history source of truth

## 还没确认的点

1. `codex-rollout` 自身的物理文件组织和 materialization 机制还没单独看。
2. `state` 与 `rollout` 的桥接面需要再追 `state_db_bridge` / rollout crate。
3. memory/memories 相关 job state 是否会进一步影响恢复路径，还没展开。

## 下一步建议

继续写：
- `state crate 在系统里的职责`
- `session / thread / rollout 恢复链`
- `config layer 与 trust / project layer` 详细版
