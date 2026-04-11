# Codex 源码拆解 28：analytics 与 telemetry 归约链

## 这篇看什么

回答：**Codex 的 analytics 不是简单打点，它到底怎么从事实输入、状态归约，最后变成可发送事件？**

## 先给结论

analytics 这块的核心设计是：

> **fact ingestion → reducer state enrichment → wire event request**

它不是到处直接 `POST event`，而是分三层：

1. fact
   - initialize / response / request / notification / custom
2. reducer
   - 缓存 connection / thread metadata
   - 把轻量事实补全成可发事件
3. client queue
   - 后台任务异步 POST

一句话：

> analytics 的重心不是“发请求”，而是“先把零散事实归约成稳定事件模型”。

## 关键代码路径

- `codex-rs/analytics/src/client.rs`
- `codex-rs/analytics/src/facts.rs`
- `codex-rs/analytics/src/reducer.rs`
- `codex-rs/analytics/src/events.rs`
- `codex-rs/app-server/src/message_processor.rs`
- `codex-rs/app-server/src/codex_message_processor.rs`

## 总体架构

### client 层
`AnalyticsEventsClient` 暴露：
- `track_initialize`
- `track_response`
- `track_skill_invocations`
- `track_app_mentioned`
- `track_app_used`
- `track_plugin_used/install/enable/disable`
- `track_compaction`
- `track_subagent_thread_started`
- `track_guardian_review`

这里看上去像一堆埋点 API，但它们并不直接发送，而是先进 queue。

### facts 层
`AnalyticsFact` 是统一 ingestion 面：
- `Initialize`
- `Response`
- `Request`
- `Notification`
- `Custom`

其中很多业务事件其实是 `CustomAnalyticsFact`。

### reducer 层
`AnalyticsReducer` 维护：
- `connections`
- `thread_connections`
- `thread_metadata`

它的作用是：
- 把 thread 关联到 connection
- 把 client/runtime metadata 和 thread/source/subagent 信息拼起来
- 再决定是否能发最终事件

## app-server 和 core 的分工

### app-server
主要提供：
- initialize
- thread start/resume/fork 这种 connection/thread 生命周期事实

也就是说，app-server 是 analytics 的：

> 连接态 / 控制面元数据来源

### core
主要提供：
- skills
- plugin/app 使用
- compaction
- subagent thread started
- 一些 guardian 相关的业务事实

也就是说，core 是 analytics 的：

> 行为态 / 产品态事实来源

这个分层很合理。

## reducer 最关键的价值

很多事件并不是“拿到就能发”。

例如：
- guardian review
- compaction
- plugin/app usage

往往都需要知道：
- 这个 thread 属于哪个 connection
- app-server client 是谁
- thread source 是 user 还是 subagent
- subagent source 是 review / compact / memory_consolidation 之类

所以 reducer 的意义是：

> 用 earlier lifecycle facts 补全 later business facts。

这就是为什么 initialize 本身可能不发事件，只是先 seed state。

## skill/plugin/app 的归一化

### skills
skill 不是直接用 path 字符串当 id，而是：
- repo scope → 相对 repo root
- user/admin/system → 绝对 path
- 再做稳定 `skill_id` 计算

所以 skills analytics 明显考虑了：
- repo portability
- personal/global path 稳定性

### plugins
plugin metadata 会归约成：
- plugin id/name/marketplace
- `has_skills`
- `mcp_server_count`
- `connector_ids`
- `product_client_id`

这说明 analytics 不是只想知道“用了哪个 plugin”，而是想知道：

> 这个 plugin 是什么类型的能力 bundle。

### guardian / subagent
`thread_source_name()` / `subagent_source_name()` 会把 session/subagent source 归一化成：
- `user`
- `subagent`
- `review`
- `compact`
- `thread_spawn`
- `memory_consolidation`
- ...

这说明 analytics 其实已经在为“多来源 agent 工作流”建统一维度。

## dedupe / gating
analytics 不是无脑全发。

### dedupe
- `app_used` 按 `(turn_id, connector_id)` 去重
- `plugin_used` 按 `(turn_id, plugin_id)` 去重

### gating
- `analytics_enabled == false` 时直接不记
- app-server 还会受 `Feature::GeneralAnalytics` 控制
- core 某些路径也会再看 feature flag

这说明 analytics 设计偏谨慎：
- 避免风暴
- 避免重复
- 避免在缺上下文时发脏数据

## 一个重要发现
`track_guardian_review()` 的 schema/reducer/client 都在，
但实际生产调用点这轮没看到很明确的 wire-up。

这意味着：
- guardian analytics 可能还没完全接通
- 或只接了别处，这次没挖到

这个要在后面写 guide 时注明，别说满。

## 当前边界判断

- `facts`：轻量输入面
- `reducer`：上下文补全与事件归约层
- `events`：最终 wire schema
- `client queue`：异步发送层
- app-server：连接态元数据来源
- core：行为态事实来源

## 设计判断

### 1. 这套设计是成熟的
比“到处直接发埋点请求”高一个层级。

### 2. reducer 是这个系统的真正核心
没有 reducer，很多事件都只有半截信息。

### 3. plugin/skill/app/guardian/subagent 都在往统一 telemetry 语言收敛
这说明 OpenAI 已经把 Codex 看成复杂 agent workflow 系统，而不是单线程聊天器。

## 还值得继续观察的点

1. guardian review analytics 的真实发射点
2. request/notification fact 为何当前 reducer 基本没怎么消费
3. telemetry 是否未来会进一步和 app-server protocol surface 对齐
