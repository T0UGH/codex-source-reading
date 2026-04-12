---
title: Codex guidebook 10｜realtime、collab 与 memories / migration 专题
date: 2026-04-12
status: draft
---

# 10｜realtime、collab 与 memories / migration 专题

## 先给结论

到这里，Codex 的主骨架已经讲完。剩下这一批最值得补的，不再是主 runtime spine，而是几条会明显影响系统形态的扩展线：

- realtime conversation
- collab / multi-agent control plane
- memories startup pipeline
- external-agent-config migration layer

这几条线如果分别写，容易太碎；如果全部塞进第 6 章，又会把 capability 与高级系统那章撑得太满。

所以更合适的收尾方式，是单独拉一篇专题，把它们统一放在一个主题下理解：

> **它们都是“让 Codex 超出单线程对话 agent 形态”的扩展系统。**

其中可以再分成两半：

1. **live control plane**：realtime + collab/multi-agent
2. **startup adaptation layer**：memories + external-agent-config

---

## Part A：live control plane

## 1. realtime 和 collab 为什么值得放在一起讲

realtime 和 collab 看起来属于完全不同的产品面：

- 一个更像实时语音/实时会话
- 一个更像多 agent 协作

但从系统角色看，它们有一个共同点：

> **都在 session 边界外，再挂出一套 live control plane。**

也就是说，它们不是简单“再多几个工具”，而是在现有 turn/runtime 主链之外，再建立额外的运行控制面。

所以把它们放在一起，不是因为它们功能相似，而是因为它们在架构里的层级相似。

---

## 2. realtime：会话级实时控制面，而不是普通消息流扩展

`RealtimeConversationManager` 的存在说明，Codex 对 realtime 的理解不是“给普通 session 再加个 websocket”，而是：

> **为一个 session 建一套独立的 live transport/control runtime。**

它会持有：

- 当前 active realtime conversation state
- 音频输入输出相关通道
- user text 通道
- writer
- handoff state
- input task
- fanout task
- realtime active 状态

这意味着 realtime 不是薄层接口，而是一套 session-local live runtime。

而且它还有一个很强的边界：

- 同一时刻只允许一个 active realtime conversation
- 新开会替掉旧的

这进一步说明它不是一个随手附加的 feature，而是一条正式的会话控制面。

---

## 3. realtime 的关键控制对象：不是 transport，而是 ConversationState

如果只从网络角度看 realtime，会很容易把重点放在 websocket / webrtc 上。

但从架构层面，真正重要的不是 transport 类型，而是 `ConversationState` 这种会话控制对象。

transport 可以不同，但系统真正关心的是：

- 当前 conversation 有没有 active
- 输入从哪里来
- 输出往哪里去
- handoff 正在做什么
- 什么时候可以发下一个 response.create

所以更准确地说：

> **realtime 的核心不是“选哪种传输”，而是“如何维持一个活的实时会话状态机”。**

这也是为什么这条线应该在 guidebook 里被看成 runtime control plane，而不是 transport feature。

---

## 4. `RealtimeResponseCreateQueue` 为什么重要

这条线上最能体现“控制面”味道的一个点，是 response create queue。

原因很简单：某些 realtime backend 并不允许你无限制同时发多个默认 response。如果系统不做调度，就会出现响应创建冲突。

Codex 的做法不是把这个复杂性交给上层，而是在本地放了一个很小但关键的对象：

- 当前是否已有 active response
- 如果有，新 create 请求先记为 pending
- 等 response done/cancel 后再 flush
- 如果后端返回 active-response 已存在，也走 defer 而不是直接炸掉

这说明系统在做的不是“收消息就转发”，而是：

> **把 backend 约束吸收进本地控制面。**

这正是一个成熟 live runtime 的做法。

---

## 5. handoff：realtime 和普通 turn 引擎的桥

realtime 这条线最有意思的地方，在于它并没有把自己完全做成一套平行世界。

相反，它通过 handoff 把自己接回普通 Codex session 引擎：

- realtime backend 可以发出 `HandoffRequested`
- fanout loop 提取文本
- 再调用 `Session::route_realtime_text_input`
- 最终把这段内容重新注入普通 `Op::UserInput` 主链

这说明 realtime 虽然有自己的 live control plane，但它并没有试图重新发明整套任务/turn/runtime 系统，而是在关键时刻选择回到普通主线。

所以 handoff 的真正意义是：

> **realtime 不替代主 turn engine，而是在需要时把内容重新归入主 turn engine。**

---

## 6. collab / multi-agent：另一套 rooted thread-tree control plane

和 realtime 平行的另一条 live control line，是 multi-agent collaboration。

这里最关键的对象不是 tool handler，而是 `AgentControl`。

它的定位非常明确：

> **root-thread/session-tree scoped control-plane handle。**

也就是说，它不是全局 agent manager，而是一棵以当前 root thread 为中心的控制面。

它负责的事情包括：

- spawn agent
- fork thread
- message / interrupt / wait / close
- registry 里的 agent 身份和路径
- session tree 内部的生命周期控制

所以 multi-agent 这条线真正的主人不是某个 `/spawn` 工具，而是 `AgentControl` 这套 control plane。

---

## 7. 为什么 `multi_agents_v2` handlers 只是 facade

如果只读工具处理器，容易觉得 collab 的主逻辑都在 handler 里。

但从职责上看，这些 handlers 更像前端门面：

- 解析参数
- 发 begin/end event
- 调 `AgentControl`

真正的线程树控制逻辑、fork 逻辑、metadata reservation、spawn edge persistence、shutdown 等关键行为，都在 control layer 里。

所以这条链最稳的判断是：

> **multi-agent v2 handlers 是 façade，`AgentControl` 才是真正控制面。**

这和前面的 app-server facade / core owner 关系，其实是同一种架构风格再现。

---

## 8. collab 为什么要围绕 thread tree 组织，而不是 agent list

Codex 的 agent 协作不是一组松散的 worker，而是一棵 rooted thread tree。

这体现在：

- root 被显式识别
- child thread 可以 fork 或 fresh spawn
- agent 有 canonical path
- 关闭 agent 不只是停一个进程，还要关闭 subtree 并更新 spawn edge 状态

所以 collab 这套系统更像：

> **thread-tree orchestration**

而不是单纯“起多个 agent 会话”。

这也解释了为什么 app-server 对 collab 的投影，不只是消息列表，而是更像一组带生命周期的 `CollabAgentToolCall` items。

---

## 9. realtime 与 collab 的共同点和差别

这两条线可以这样对照。

### 共同点
- 都是 live control plane
- 都不是主 turn spine 本身
- 都要把复杂 runtime 状态投影给 app-server / UI
- 都在 session 边界外多挂一层系统

### 差别
#### realtime
- 面向实时会话
- 关键对象是 conversation state
- 重点是 transport、response scheduling、handoff

#### collab
- 面向多 agent 协作
- 关键对象是 thread tree / agent registry / control plane
- 重点是 spawn、fork、message、close、wait

所以它们不是一回事，但放在同一篇专题里是合理的：

> **一个是 live conversation plane，一个是 live collaboration plane。**

---

## Part B：startup adaptation layer

## 10. memories 为什么不像普通 feature，而像 pipeline

memories 这条线如果只看名字，很容易把它误会成“记忆工具”或“多一点 prompt 注入”。

但从源码结构看，它明显是一条 startup pipeline：

- 有统一公开入口
- 明确分 phase 1 / phase 2
- 依赖 state runtime 的 durable job 协调
- 会在启动时判断当前 session 是否有资格触发
- 有 singleton global consolidation job

所以更合理的定位是：

> **启动期长期记忆提取与整合管线。**

它不是普通 turn-time helper，而是一个后台启动流程系统。

---

## 11. memories 为什么要拆 phase 1 / phase 2

这条管线拆成两段不是形式主义，而是职责真的不同。

### phase 1
- 面向多个线程
- 从 stale eligible threads 里提取 stage1 memories
- 结果写入 DB
- 成功后触发 phase 2 的全局 job 更新

### phase 2
- 面向全局 consolidation
- 从 stage1 outputs 里选输入集
- 同步 filesystem working set
- 启一个受限内部 subagent 做整合
- 成功后更新 selection baseline

这说明 memories 不是“每个线程自己做完就结束”，而是：

> **先 thread-level extract，再 global-level consolidate。**

这套分层非常清楚，也很像正式数据管线，而不是 prompt trick。

---

## 12. state runtime 才是 memories 工作流的真正协调器

memories 里最值得注意的一点，是 phase 逻辑虽然写在 core 里，但真正协调 durable workflow 的是 state runtime。

这里面管的不是“怎么问模型”，而是：

- job claim
- watermark
- retry/backoff
- retention
- selected baseline
- singleton global phase2 lock

所以更合理的架构判断是：

- core memories 模块负责 orchestration
- state runtime 负责 durable workflow coordination

这也是为什么 memories 不是一小段启动脚本，而是一条真正的 pipeline。

---

## 13. phase 2 为什么要用受限内部 agent 来做

phase 2 的另一个设计亮点，是它不是自己在本地代码里直接做 consolidation，而是启动一个内部 agent 来做。

但这个 agent 又不是普通 agent。它的 config 被明显收紧：

- cwd 限在 memory root
- `generate_memories = false`
- approvals = never
- 禁用 `SpawnCsv`、`Collab`、`MemoryTool`
- workspace-write 但无网络
- 指定专用 model / reasoning effort

所以这件事说明的不是“Codex 到处都用 agent”，而是：

> **Codex 会把自己的 agent runtime 当成内部 worker 来复用，但会给它套上一个强约束包壳。**

这是一个很成熟的系统设计姿态。

---

## 14. external-agent-config 为什么更像 migration layer

最后看 external-agent-config。

它和 memories 放在一起，不是因为两者功能相关，而是因为它们都属于“启动时适配外部环境”的系统。

external-agent-config 的职责很清楚：

- detect 外部 agent/config 资产
- import 到 Codex 的结构里
- 尽量以 additive merge 的方式做迁移
- 重写术语和目录位置

从当前实现看，它明显偏向：

- Claude / 现有外部生态配置迁移
- 本地 skills / AGENTS/CLAUDE 文档转接
- config additive import

所以最准确的定位是：

> **migration / compatibility layer**

而不是主 runtime 子系统。

---

## 15. external-agent-config 有一个明显“架子先搭好、行为还没补全”的点

这条线还有个很适合写进专题的实现状态判断：

- 类型系统和 API 表面上已经有 `McpServerConfig` migration item
- 但 detect/import 的行为实现里，这块还没真正落下去

也就是说，这里明显存在一种状态：

> **接口先暴露了，真正迁移逻辑还没补完。**

这类“架子已搭、行为未全”的点很有价值，因为它能帮助后续读源码的人判断：

- 这是方向已经定了、实现未补齐
- 不是压根没有设计过

---

## 16. 这一章最重要的判断

读完这章，应该稳定这些结论。

### 判断 1：realtime 和 collab 都是 live control plane
不是普通 feature。

### 判断 2：realtime 的关键对象是 conversation state
transport 只是实现细节之一。

### 判断 3：handoff 是 realtime 回接主 turn 引擎的桥
realtime 没有自成平行世界。

### 判断 4：collab 的真正 owner 是 `AgentControl`
`multi_agents_v2` handlers 只是 façade。

### 判断 5：multi-agent 是 rooted thread-tree control plane
不是松散 worker 列表。

### 判断 6：memories 是 startup pipeline
不是 turn-time helper。

### 判断 7：phase 1/phase 2 对应 extract 与 global consolidate
这是一条真正的两阶段管线。

### 判断 8：state runtime 是 memories durable workflow coordinator
不是可有可无的 sidecar。

### 判断 9：external-agent-config 是 migration layer
不是主 runtime spine 的一部分。

### 判断 10：`McpServerConfig` migration 目前更像未完成接口
类型在，行为未全。

---

## 17. 到这里，这一批专题补了什么

这批追加专题补完之后，仓库里已经不仅有主 guidebook spine，还有第二层更细的专题：

- model transport / backend-client
- review / guardian
- realtime / collab
- memories / migration

也就是说，接下来如果还要继续推进，最合理的方向已经不是再继续正文铺新章，而是：

- 做 appendices
- 做关键函数索引
- 做 open questions
- 或对某一个专题再继续函数级下钻

这时仓库已经从“还缺大块主干”进入了“可持续扩展的专题层”。

---

## 关键文件

- `codex-rs/core/src/realtime_conversation.rs`
- `codex-rs/core/src/agent/control.rs`
- `codex-rs/core/src/tools/handlers/multi_agents_v2/spawn.rs`
- `codex-rs/core/src/tools/handlers/multi_agents_v2/message_tool.rs`
- `codex-rs/core/src/tools/handlers/multi_agents_v2/wait.rs`
- `codex-rs/core/src/tools/handlers/multi_agents_v2/close_agent.rs`
- `codex-rs/app-server/src/bespoke_event_handling.rs`
- `codex-rs/core/src/memories/mod.rs`
- `codex-rs/core/src/memories/start.rs`
- `codex-rs/core/src/memories/phase1.rs`
- `codex-rs/core/src/memories/phase2.rs`
- `codex-rs/state/src/runtime/memories.rs`
- `codex-rs/core/src/external_agent_config.rs`
- `codex-rs/app-server/src/external_agent_config_api.rs`
