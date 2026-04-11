# Codex 源码拆解 24：memories、agents 与 external-agent-config

## 这篇看什么

回答：**Codex 的 memories 子系统怎么跑？agent runtime 在系统里是什么地位？`external-agent-config` 又到底在做什么？**

## 先给结论

这是三块经常被忽略、但长期很关键的系统：

1. `memories`
   - 是一个 **启动期两阶段 memory generation pipeline**
2. `agents`
   - 是一个 **session-scoped 的多 agent 控制平面**
3. `external-agent-config`
   - 当前主要是 **把 Claude 配置/技能/说明迁到 Codex** 的迁移器

一句话：

> memories 负责长期提炼，agents 负责运行时协作，external-agent-config 负责跨 agent 生态迁移。

## 关键代码路径

- `codex-rs/core/src/memories/*`
- `codex-rs/state/src/runtime/memories.rs`
- `codex-rs/core/src/agent/*`
- `codex-rs/protocol/src/agent_path.rs`
- `codex-rs/core/src/external_agent_config.rs`
- `codex-rs/app-server/src/external_agent_config_api.rs`

## memories：不是 prompt 小功能，而是正式 pipeline

### Phase 1
- 选择 eligible historical threads
- claim per-thread jobs
- 跑 extraction
- 存 `stage1_outputs`

### Phase 2
- claim 一个全局 consolidation job
- 选 top stage1 outputs
- 重建 artifacts
- 再起一个 consolidation subagent

也就是说，memories 不是：
- 实时边聊边存一句摘要

而是：
- **启动期批处理 + consolidation agent**

这非常工程化。

## memory 存储是双层的

### SQLite state
存：
- stage1 outputs
- jobs
- phase2 selection snapshot
- usage tracking 等

### filesystem artifacts
存：
- `memories/` 目录下的 summary / raw outputs 等文件

所以 memory 这块也延续了 Codex 常见的设计：

> 结构化状态进 DB，最终可读 artifact 落文件。

## memory read path 和 generation path 是分开的

### generation path
- phase1/phase2 startup pipeline

### read path
- 正常 turn 时把 `memory_summary.md` 截断后注入 developer instructions

这说明 memories 不是单一模块，而是：
- 后台生成
- 前台消费

两条线分离。

## agents：不是“能 spawn 一个子线程”这么简单

`AgentControl` 已经很清楚是一个 session-scoped control plane：

- 保留 spawn slot
- 分配 nickname/path
- spawn child thread
- 跟踪状态
- 支持 path-based resolve/list

这说明 Codex 的多 agent 不是靠“临时记录几个 thread id”实现的，而是有一套正式 registry/control 结构。

## agent path 是 v2 多 agent 的关键升级

现在已经不是只靠 thread id。

`AgentPath` 支持：
- `/root/...` 绝对路径
- 相对路径 resolve
- session root 注册

这说明 OpenAI 已经把多 agent 从“多几个 thread”升级成：

> 有层级、可寻址的 agent tree。

这个方向很重要。

## agent roles 是 config-layer overlay

agent role 可以来自：
- built-in
- `[agents.roles]`
- config-layer `agents/` 目录
- standalone role TOML

spawn 时会把 role overlay 到 config 上。

而且 role 应用时还会尽量保留 caller 选好的 profile/provider，除非 role 明确接管。

这说明 agent role 设计不是“硬模板替换”，而是偏策略叠加。

## external-agent-config：当前主要是 Claude → Codex 迁移器

这个名字很泛，但当前实现很具体：

- 检测 `~/.claude/settings.json`
- 检测 `~/.claude/skills`
- 检测 `CLAUDE.md`
- repo 级也会看 `.claude/...`

导入目标则是：
- Codex config
- `.agents/skills`
- `AGENTS.md`

而且会把文本里的：
- Claude
- Claude Code

等术语改写成：
- Codex

这说明它现在更像：

> Claude 生态迁移适配器

而不是一个完全抽象的多外部 agent importer。

## Claude config 导入是有选择的，不是全量搬运

当前主要映射包括：
- `env` → shell environment policy
- `sandbox.enabled = true` → `workspace-write`

并且：
- 只补缺失字段
- 不会粗暴覆盖已有 Codex config

这说明迁移策略是偏保守、增量式的。

## memories / agents / external-agent-config 三者怎么放在大图里

### memories
是长期经验层。

### agents
是运行时协作层。

### external-agent-config
是跨生态迁移层。

它们不是一条链，但共同说明了 Codex 在往三个方向延展：
- 长期状态
- 多 agent 协作
- 外部生态接入

## 当前边界判断

- memories：启动期 memory generation pipeline + runtime read injection
- agents：session-scoped multi-agent control plane
- agent path：多 agent 可寻址层
- agent role：spawn-time config overlay
- external-agent-config：当前以 Claude 迁移为主的导入器

## 设计判断

### 1. memories 已经是正式后台系统，不是 prompt 小玩具

### 2. agents 是 Codex 未来架构里很核心的控制面能力
这块不是附带能力，而是会越来越重要。

### 3. external-agent-config 的命名比当前实现更泛
实现目前明显是 Claude-first，这点写 guide 时要说实话。

## 还值得继续观察的点

1. `McpServerConfig` migration item 目前看像是占位，后续是否会真正接入
2. memories 的 consolidate agent prompt / artifact 产出还可以单独继续拆
3. agent analytics / agent tree UI 呈现还值得后续专门补一篇
