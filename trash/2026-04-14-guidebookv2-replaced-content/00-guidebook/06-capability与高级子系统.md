---
title: Codex guidebook 06｜capability 与高级子系统
date: 2026-04-12
status: draft
---

# 06｜capability 与高级子系统

## 先给结论

到了 guidebook 的最后一章，最容易犯的错，就是把剩下的名词都当成“边角功能”堆在一起。

但从源码关系看，更好的整理方式不是机械列目录，而是把它们分成两半：

1. **capability substrate**：决定一个 turn 能用什么能力
2. **higher-order systems**：决定系统如何协作、审查、记忆和扩展 session 行为

所以这章的核心结论可以先写死：

> **plugin、skills、MCP、apps、connectors 解决的是“这个 turn 能接到什么能力”；review、guardian、realtime、collab、memories、agents 解决的是“这个 runtime 怎么扩展成更高阶的工作系统”。**

这两半不该混成一个功能杂烩，但也没必要拆成两个弱章节。把它们合成一章，更容易把 Codex 的“扩展面”收干净。

---

## Part A：capability substrate

## 1. plugin 为什么是更高一层的能力打包单元

理解 capability system 的第一步，不是先看 skills 或 MCP，而是先看 plugin。

从源码角色看，plugin 更像：

> **共享能力打包与激活单元。**

它本身不一定直接执行什么 runtime 行为，但它会把很多能力源组织起来，然后在启动和运行时导出：

- skills root
- MCP server names
- apps/connectors 相关元数据
- plugin instructions / summaries

这意味着 plugin 的作用，不是一个简单的“插件文件夹”概念，而是：

- 让能力声明有统一打包层
- 让后续运行时可以从 config/layer stack 派生出可用能力图

所以 capability system 的总入口，更适合从 plugin 来理解，而不是从单个 skill 或单个 MCP 配置来理解。

---

## 2. skills 是行为注入层，不是 transport 层

skills 在 Codex 里不是外部协议，也不是执行后端。它更接近：

> **行为指导与依赖要求注入层。**

它负责的是：

- 提供额外的行为/任务指导
- 暴露依赖要求
- 让 turn 在特定上下文下具备某种“应该怎么做”的能力

所以 skills 的价值不在 transport，而在 behavior。

如果把 skills 和 MCP 混成一回事，就会看不清楚：

- MCP 解决的是外部工具怎么接进来
- skill 解决的是模型/agent 在当前任务里该如何利用这些能力

这也是为什么它们虽然会发生依赖关系，但本质不是同一层东西。

---

## 3. MCP 是外部能力 transport/config 层

MCP 这一侧，真正的重点不是 prompt，而是：

- server 配置
- transport
- auth
- recovery
- runtime 上怎样把 MCP 工具纳入有效能力图

所以更准确地说，MCP 在这里解决的是：

> **外部工具能力如何通过统一协议接入到 Codex runtime。**

这也是为什么：

- `mcp-server` 是对外暴露 Codex 的一条线
- `rmcp-client` / `McpManager` 是把外部 MCP 能力接进来的另一条线

这两条线都和 MCP 有关，但一个是 outward exposure，一个是 inbound capability integration。

---

## 4. skill 和 MCP 的关系：并行能力线，而不是主从关系

一个常见误解是：skill 是 MCP 的上层包装，或者 MCP 是 skill 的底层实现。

源码关系更稳的判断是：

> **skills 和 MCP 是两条并行能力线。**

它们最强的耦合点，不是类型系统上的层级依赖，而是：

- plugin packaging
- config derivation
- skill 对 MCP server 的依赖声明
- runtime 初始化阶段的共同组装

换句话说：

- skill 可以依赖 MCP
- 但 skill 不是 MCP 的语义外壳
- MCP 也不是 skill 的实现细节

这种并行关系，比“上层/下层”关系更符合源码事实。

---

## 5. apps 和 connectors：不是单独一张表，而是多来源合流

apps/connectors 这块也很容易被看成“某个目录里的产品能力列表”。

但从组合方式上看，它更像一个融合层。它会把这些来源合起来：

- 目录态 metadata
- codex-apps MCP 可访问工具
- plugin 声明
- runtime 可用性/启用状态

也就是说，apps/connectors 真正做的不是“保存一个 app 列表”，而是：

> **把目录信息、可访问能力和 plugin 声明合并成一个用户可感知的 capability 视图。**

所以 apps/connectors 应该被放在 capability substrate 里理解，而不是孤立看成产品壳。

---

## 6. capability system 的一句话总结

到这里，可以把 capability 这半边系统压缩成一句话：

> **plugins 负责打包，skills 负责指导，MCP 负责接入，apps/connectors 负责产品化呈现。**

这四者一起回答的是：

> **一个 turn 到底能用什么。**

---

## Part B：higher-order systems

## 7. `/review` 和 guardian 不是一回事

进入高级子系统后，第一个必须拆开的误区就是 review 和 guardian。

### `/review`
它更像一个明确的用户工作流：

- 用户显式发起 review
- 系统会进入专门的 reviewer session / reviewer prompt / reviewer config 路径
- 它是一个产品能力

### guardian
guardian 更像 approval reviewer infrastructure：

- 它处理审查、批准、复用、子会话等问题
- 它更接近 approval/review 基础设施
- 不是单纯的“给用户看的 review 命令”

所以更合理的关系不是“review=guardian”，而是：

> **review 是产品工作流，guardian 是审查基础设施。**

这也是为什么之前那篇 `review与guardian链路` 很重要：它拆开的就是这两个层级。

---

## 8. realtime 和 collab 也不是一回事

这组名字同样容易被看成一套功能。

### realtime
它更像：

- live conversation / live transport / 实时会话状态
- 一套面向实时互动的运行面

### collab
它更像：

- 多 agent 协作
- 任务分配
- 子 agent 生命周期
- session 内协同控制面

所以更合适的理解是：

- realtime 解决“实时对话与实时状态”
- collab 解决“多 agent 协作运行”

两者都属于高级 runtime systems，但不是同一个子系统。

---

## 9. memories 是启动期管线，不是普通会话 feature

memories 这块如果只看功能名，很容易觉得它只是另一个 prompt enhancement。

但从模块职责看，它更接近：

> **启动期的长期记忆整理与注入管线。**

它有明确的 phase 划分，会做：

- memory 相关状态处理
- summarization / consolidation
- DB / filesystem 之间的同步

所以 memories 不是“会话中随手调个 helper”，而是一个更偏 startup pipeline 的系统。

这也解释了为什么它应该和 agents、collab、review 这些 session-time systems 区分开。

---

## 10. agents 是 session-scoped control plane

如果说 collab 是多 agent 协作行为面，那么 `AgentControl` 这一套更像：

> **session-scoped multi-agent control plane。**

它处理的是：

- spawn
- reservation
- inheritance
- 子 agent 路径 / 元数据
- 多 agent 生命周期控制

这说明 Codex 里的 agents 不是一组孤立功能，而是一套正式控制面。

所以在这章里，agents 应该被看成高级系统的骨架层，而不是某个局部能力点。

---

## 11. external-agent-config 更像迁移层，不是章节支柱

external-agent-config 也值得提，但它不应该和前面几条主线并列成一根大柱子。

原因在于：它当前更偏：

- 外部 agent 生态适配
- 迁移 / 导入
- 兼容某些现有外部配置体系

尤其是从当前实现看，它明显带有较强的迁移和兼容意味。

所以在这章里，它更适合被放成：

- 生态兼容
- migration shim
- 附属说明

而不是 capability system 或 advanced runtime 的主轴之一。

---

## 12. 为什么这章要把 capability 和高级子系统合并

一开始 guidebook 设计里，capability 和高级子系统原本是两章候选。

但从实际源码关系和正文写作角度看，把它们合成一章更合理，原因有三个：

### 原因 1：它们都不是单条主 runtime spine
和 app-server、turn-history、unified-exec 不同，这两块更像系统的扩展面，而不是单一主链。

### 原因 2：它们都通过 session/runtime assembly 挂到系统里
插件、skills、MCP、apps 会影响 turn capability graph；review、guardian、collab、agents、memories 会影响 session/runtime extension graph。

它们最终都在回答同一类问题：

> **Codex 除了主执行主链以外，还能扩到哪里、怎么扩。**

### 原因 3：拆成两章容易都写薄
如果强行拆开，容易变成：

- 一章像 capability 名词表
- 一章像高级功能清单

合成一章，反而更容易形成一个明确结尾：

> **前面五章讲主骨架，这一章讲扩展面。**

---

## 13. 这一章最重要的判断

读完这章，应该稳定这些结论。

### 判断 1：plugin 是更上层的能力打包单元
很多 capability 的汇合点都在它上面。

### 判断 2：skills 是行为注入层
不是 transport 层。

### 判断 3：MCP 是外部能力接入层
它和 skill 是并行能力线，不是上下级关系。

### 判断 4：apps/connectors 是多来源能力融合层
不是一个简单的目录表。

### 判断 5：review 和 guardian 不一样
前者是产品工作流，后者是审查基础设施。

### 判断 6：realtime 和 collab 不一样
前者偏实时会话，后者偏多 agent 协作运行。

### 判断 7：memories 更像 startup pipeline
不是普通 session feature。

### 判断 8：agents 是 session-scoped control plane
不是零散功能集合。

### 判断 9：external-agent-config 更像迁移兼容层
不该被抬成主骨架章节。

---

## 14. 到这里，这套 guidebook 完成了什么

读完前面六章，应该已经能稳定回答这些问题：

- Codex 的主体在哪一层
- 状态和恢复如何成立
- app-server 为什么不是 runtime owner
- turn history 为什么不是 event log 镜像
- unified-exec 为什么是独立子系统
- capability 与高级系统如何挂进整体 runtime

也就是说，这套导读到这里，已经基本把 Codex 的主架构骨架闭合起来了。

后面再往下走，更适合进入：

- 附录
- 关键函数索引
- call-chain 索引
- open questions
- 以及更细的函数级共读回溯

---

## 相关阅读

- 想继续按专题深挖 capability / transport 边界：[[07-model-client与provider请求主链]]、[[08-codex-client-codex-api与backend-client分层]]
- 想继续深挖 review / guardian：[[09-review工作流与guardian审查基础设施]]
- 想继续深挖 realtime / collab / memories：[[10-realtime-collab与memory迁移专题]]
- 想从能力与高级系统跳到函数/调用链层：[[11-关键函数索引]]、[[12-调用链索引]]

## 关键文件

- `codex-rs/core/src/plugins/manager.rs`
- `codex-rs/core/src/skills.rs`
- `codex-rs/core/src/mcp.rs`
- `codex-rs/core/src/mcp_skill_dependencies.rs`
- `codex-rs/core/src/connectors.rs`
- `codex-rs/app-server/src/codex_message_processor/plugin_app_helpers.rs`
- `codex-rs/core/src/tasks/review.rs`
- `codex-rs/core/src/guardian/review_session.rs`
- `codex-rs/core/src/realtime_conversation.rs`
- `codex-rs/core/src/agent/control.rs`
- `codex-rs/core/src/memories/mod.rs`
- `codex-rs/core/src/memories/phase2.rs`
- `codex-rs/state/src/runtime/memories.rs`
- `codex-rs/core/src/external_agent_config.rs`
