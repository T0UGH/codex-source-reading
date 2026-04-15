---
title: Codex 卷六写作卡片
date: 2026-04-13
status: draft
tags:
  - Codex
  - guidebook
  - writing-cards
  - volume-6
  - review
  - guardian
  - collab
  - memories
---

# Codex 卷六写作卡片

> 用途：作为 Codex 新卷六《审查、协作与高级 runtime 为什么不能当作边角功能》的内部写作卡片。先按卷级主问题搭骨架，再从 `00-guidebook/09`、`00-guidebook/10`、`00-guidebook/15`、相关 `01-source-notes/` 和源码锚点抽素材，禁止让旧专题稿的顺序直接决定新卷六结构。

---

## 卷六总问题

这一卷要回答的是：

> **为什么 review、guardian、realtime、collab、memories、agents 这些看起来像“高级功能”的东西，其实代表 Codex 正在长出更高层的运行组织能力？**

这一卷真正要立住的是：

- `/review` 和 guardian 不是一回事
- guardian 更像审查基础设施，而不是一条普通功能链
- realtime / collab / AgentControl 不是一层东西，而是在把 Codex 推向多主体协作 runtime
- memories 更像启动管线和长期组织能力，而不是普通 turn-time helper
- 这些系统合在一起，说明 Codex 后半段不只是“多几个高级功能”，而是在长更高层的 runtime 组织面

一句话压缩：

> **卷六负责把 Codex 作为一套“开始长出审查、协作与更高层运行组织能力的系统”讲明白。**

这卷不是 feature catalog，也不是“高级功能合集”，更不是把前几卷没讲完的内容一股脑塞进来。

---

## 建议篇章方向（v1）

### 01｜为什么 `/review` 和 guardian 不是一回事

#### 主问题
很多人第一次看到 review 相关源码时，会把 `/review` 工作流和 guardian 审查链理解成同一件事。为什么这个理解会把卷六入口直接读歪？

#### 核心判断句
**`/review` 更像用户可触发的 review workflow，而 guardian 更像 approval / 审查基础设施；两者相关，但不是同一层系统。**

#### 这篇必须完成的任务
- 切开 review workflow 与 guardian infrastructure
- 让读者知道卷六不是“再讲一遍控制面”或“再讲一遍执行链”
- 把卷六入口从“功能名相似”提升到“系统职责不同”

#### 这篇不讲什么
- 不提前展开 guardian analytics 缺口
- 不提前进入 realtime / collab / memories
- 不把这篇写成命令用法说明

#### Mermaid 主图
1. `/review` workflow vs guardian infrastructure 分层图
2. user-facing review path / internal guardian path 对照图

#### 需要参考的旧文档（绝对路径）
- `~/workspace/codex-source-reading/00-guidebook/09-review工作流与guardian审查基础设施.md`

#### 需要参考的 source notes
- 原 22

#### 对后文导流
- 第 02 篇进入：guardian 为什么更像审查基础设施
- 第 03 篇进入：realtime / collab / AgentControl 怎样进入更高层 runtime

---

### 02｜guardian 为什么更像审查基础设施，而不是一条普通 feature 链

#### 主问题
为什么 guardian 不该被读成“另一个高级功能”，而应该被读成一套审查基础设施？

#### 核心判断句
**guardian 的价值不在某一次审查动作，而在它试图把审查判断、协议暴露、运行时接入和后续可观测性组织成一层基础设施。**

#### 这篇必须完成的任务
- 把 guardian 从“功能”提升为“基础设施层”
- 说明 protocol/UI 观察面、analytics schema/reducer/client 与 runtime emission 之间的边界
- 为后文“高级 runtime 组织能力”铺路

#### 这篇不讲什么
- 不把全文写成 analytics 缺口专文
- 不重讲 `/review` 的用户路径
- 不提前展开 collab / memories

#### Mermaid 主图
1. guardian protocol / runtime / analytics 三层图
2. 审查动作 vs 审查基础设施 对照图

#### 需要参考的旧文档（绝对路径）
- `~/workspace/codex-source-reading/00-guidebook/09-review工作流与guardian审查基础设施.md`
- `~/workspace/codex-source-reading/00-guidebook/15-guardian-analytics为何还没真正接上.md`

#### 需要参考的 source notes
- 原 22
- 原 28

#### 对后文导流
- 第 03 篇进入：realtime / collab / AgentControl 的协作 runtime 问题
- 第 05 篇收口时回看 guardian 为什么属于更高层运行组织能力

---

### 03｜realtime、collab 与 AgentControl 分别是什么层

#### 主问题
为什么 realtime、collab、AgentControl 看起来都像“多主体/多会话协作”，但其实不是一层东西？

#### 核心判断句
**realtime 更像实时会话与交互层，collab 更像协作运行层，而 AgentControl 更像 rooted thread-tree 的多主体控制面；三者一起才构成 Codex 更高阶的协作 runtime。**

#### 这篇必须完成的任务
- 切开 realtime / collab / AgentControl 的层级
- 把“协作”从 feature 感拉到 runtime 组织能力
- 解释为什么这几层不能混成“多 agent 功能”一句话

#### 这篇不讲什么
- 不把 external-agent-config 全吃进来
- 不重讲卷三控制面
- 不提前进入 memories 启动管线

#### Mermaid 主图
1. realtime / collab / AgentControl 三层结构图
2. rooted thread-tree 控制面示意图

#### 需要参考的旧文档（绝对路径）
- `~/workspace/codex-source-reading/00-guidebook/10-realtime-collab与memory迁移专题.md`

#### 需要参考的 source notes
- 原 24
- 原 29

#### 对后文导流
- 第 04 篇进入：memories 为什么更像启动管线
- 第 05 篇收口：Codex 怎样长成更高层 runtime 组织结构

---

### 04｜为什么 memories 更像启动管线，而不是普通 helper

#### 主问题
为什么 memories 不该被理解成“对话时顺手调用的一点长期记忆帮助”，而更像一条启动管线和运行组织能力？

#### 核心判断句
**memories 的关键不在“多存一点信息”，而在它是启动阶段和长期组织阶段的一部分；它更接近系统如何组织长期上下文与主体能力，而不是普通 turn-time helper。**

#### 这篇必须完成的任务
- 把 memories 从 helper 心智拉到 pipeline / organization 心智
- 解释它和 agents / external-agent-config 为什么不是同一层
- 为卷六卷尾收口铺底

#### 这篇不讲什么
- 不把全文写成存储实现专题
- 不扩展到平台能力层
- 不重新解释 recovery / long-term memory 的一般概念

#### Mermaid 主图
1. startup pipeline / memories / agents 关系图
2. helper vs pipeline 对照图

#### 需要参考的旧文档（绝对路径）
- `~/workspace/codex-source-reading/00-guidebook/10-realtime-collab与memory迁移专题.md`

#### 需要参考的 source notes
- 原 24

#### 对后文导流
- 第 05 篇进入：Codex 怎样从单线程 agent 工具长成更高阶 runtime

---

### 05｜为什么说 Codex 正在长成更高层 runtime，而不是只多了一些高级功能

#### 主问题
如果把 review、guardian、realtime、collab、memories、agents 都放在一起看，为什么更准确的结论不是“Codex 多了很多高级功能”，而是“Codex 正在长成更高层 runtime”？

#### 核心判断句
**这些系统共同指向的，不是 feature list 变长，而是 Codex 已经开始长出更高层的审查、协作与运行组织结构。**

#### 这篇必须完成的任务
- 把卷六前面几篇判断压成一个 retained takeaway
- 让卷六最后收束到“运行组织能力”而不是“高级功能合集”
- 给后续平台化/更长期演化讨论留下稳定出口

#### 这篇不讲什么
- 不回头重讲 capability system
- 不写成未来规划脑暴
- 不把全文写成 open questions 索引

#### Mermaid 主图
1. review / guardian / collab / memories / agents 的高级 runtime 组织图
2. feature list vs runtime organization 对照图

#### 需要参考的旧文档（绝对路径）
- `~/workspace/codex-source-reading/00-guidebook/09-review工作流与guardian审查基础设施.md`
- `~/workspace/codex-source-reading/00-guidebook/10-realtime-collab与memory迁移专题.md`
- `~/workspace/codex-source-reading/00-guidebook/15-guardian-analytics为何还没真正接上.md`

#### 需要参考的 source notes
- 原 22
- 原 24
- 原 28
- 原 29

#### 对后文导流
- 整本手册尾段可转入：Codex 后续平台化与更高层系统演化

---

## 这一卷的固定主线顺序

> **review / guardian 边界 → guardian 基础设施层 → realtime / collab / AgentControl 协作 runtime → memories 启动与长期组织能力 → 高级 runtime 收口**

这条顺序不能随便改。原因很简单：

- 如果太早讲 collab，读者会把卷六读成“多 agent 功能卷”
- 如果太早讲 memories，读者会把卷六读成“长期记忆卷”
- 如果不先切开 `/review` 和 guardian，整卷入口就会糊
- 只有先把“这些不是一组高级 feature，而是运行组织能力”立住，卷尾才会自然成立

---

## 当前默认边界

### 这卷不应该吃掉什么
- 不吃卷五 capability system 的平台化接入与打包层问题
- 不回头重讲 unified-exec 执行会话主线
- 不把 model transport / backend-client 后置专题强塞进正文主链

### 这卷最该留下来的记忆点

> **Codex 后半段真正长出来的，不只是一些高级功能，而是审查、协作与更高层运行组织能力。**
