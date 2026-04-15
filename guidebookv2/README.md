---
title: Codex 源码导读手册 v2
date: 2026-04-15
status: draft
---

# Codex 源码导读手册 v2

> **这不是一份按模块分桶的源码笔记，而是一条连续问题链：先看清 Codex 到底是什么系统，再看它怎样跑起来、怎样持续存在、怎样把 runtime 暴露成控制面、怎样把动作落成执行会话，最后怎样长出审查、协作与更高层 runtime 组织。**

## 先读这一句

如果你把 Codex 看成：

- 一个 npm CLI
- 一个终端里的 TUI
- 一个很厚的 app-server
- 或一堆散落的 agent feature

那你读后面几卷时就会一直像在看局部零件。

这版 guidebook v2 要先修正的，正是这个错觉。

更准确的读法是：

> **Codex 先是一套以 thread runtime 为核心的 agent 系统，然后才继续长出恢复结构、控制面、执行会话系统，以及更高层的审查、协作与长期运行组织。**

所以这 6 卷不是材料归档，而是 6 个认知台阶。

---

## 这本书整体在回答什么

整本书其实只在回答一条逐层变深的问题链：

1. **Codex 到底是什么系统？**
2. **一条请求怎么真正进入 runtime 主线？**
3. **这条工作线为什么不会中断后就散掉？**
4. **runtime 怎样被暴露成稳定控制面？**
5. **动作怎样被装成可管理执行会话？**
6. **这些能力又怎样被组织成更高层 runtime？**

如果按这个顺序理解，卷与卷之间不是跳题，而是在补同一台系统的不同层。

---

## 六卷认知递进图（文字版）

### 卷一｜系统入口与总图
回答：**Codex 到底是什么，不是什么。**

这一卷负责先立整机总图：
- CLI 是入口壳
- TUI 是交互前端
- app-server 是控制面
- 真正持续推进 thread / turn 主线的是 runtime core

### 卷二｜runtime core 主线
回答：**这条工作回合到底怎么跑起来。**

这一卷把卷一 preview 过的 runtime 主体真正展开：
- 请求如何进入主线
- 当前工作面如何形成
- 动作为什么会回流当前 turn
- 一轮工作回合怎样继续与收口

### 卷三｜状态、恢复与持续工作线
回答：**为什么这条工作线不会跑完就散。**

这一卷解释：
- rollout 为什么更像正文真相源
- 恢复为什么更像 replay
- thread / turn / pending request 怎样共同组成可继续工作的工作线
- 当前工作视图怎样被认出来、被继续暴露

### 卷四｜控制面与 app-server 语义
回答：**runtime 怎样被暴露成稳定控制面。**

这一卷解释：
- app-server 为什么不是另一套平行 runtime
- listener、协议投影、状态修正怎样撑起控制面
- 控制面语义到底怎样被整理和对外暴露

### 卷五｜统一执行子系统
回答：**动作怎样从“命令”升级成“执行会话”。**

这一卷解释：
- exec.rs 与 unified-exec 为什么不是一回事
- handler、runtime、session 启动之间怎样分工
- approval、policy、sandbox、route 为什么都属于执行前主链
- transcript、process store、end event 为什么共同构成 execution control plane

### 卷六｜审查协作与高级 runtime
回答：**Codex 怎样从能执行，继续长成能组织运行。**

这一卷解释：
- `/review` 与 guardian 怎样分别长成工作流层和基础设施层
- realtime、collab、AgentControl 怎样形成多主体运行结构
- memories 为什么更像启动与长期组织管线
- 为什么这些东西不只是高级功能，而是更高层 runtime 组织能力

---

## 从哪里开始读

## 路径 A：第一次进入（推荐）
适合：第一次系统读 Codex 的读者。

建议顺序：
- [卷一](./volume-1/index.md)
- [卷二](./volume-2/index.md)
- [卷三](./volume-3/index.md)
- [卷四](./volume-4/index.md)
- [卷五](./volume-5/index.md)
- [卷六](./volume-6/index.md)

如果你只沿这一条完整顺读路径走，读完后应该能把 Codex 从整机总图一直讲到高层 runtime 组织。

## 路径 B：runtime-first
适合：你最关心 runtime 主体和持续工作线。

建议顺序：
- [卷一](./volume-1/index.md)
- [卷二](./volume-2/index.md)
- [卷三](./volume-3/index.md)

这条路径最适合先建立：
- 系统主体在哪
- 工作回合怎么跑
- 工作线为什么能持续

## 路径 C：control-plane-first
适合：你最关心 app-server、协议面、控制面暴露。

建议顺序：
- [卷一](./volume-1/index.md)
- [卷四](./volume-4/index.md)
- [卷三](./volume-3/index.md)

这条路径适合先回答：
- runtime 怎样被外层系统看见
- 状态与事件怎样被稳定投影

## 路径 D：execution-first
适合：你最关心执行链、统一执行、输出与进程控制。

建议顺序：
- [卷一](./volume-1/index.md)
- [卷二](./volume-2/index.md)
- [卷五](./volume-5/index.md)

这条路径适合先建立：
- 动作怎样从当前 turn 分叉出去
- 为什么 unified-exec 管理的是会话而不是命令

## 路径 E：high-runtime / multi-agent-first
适合：你最关心 review、guardian、collab、memories、多 agent 组织。

建议顺序：
- [卷一](./volume-1/index.md)
- [卷五](./volume-5/index.md)
- [卷六](./volume-6/index.md)

这条路径适合先建立：
- 执行会话怎样成为更高层组织的底盘
- 高层 runtime 不是 feature list，而是组织结构

---

## 当前六卷入口

1. [卷一｜系统入口与总图](./volume-1/index.md)
2. [卷二｜runtime core 主线](./volume-2/index.md)
3. [卷三｜状态、恢复与持续工作线](./volume-3/index.md)
4. [卷四｜控制面与 app-server 语义](./volume-4/index.md)
5. [卷五｜统一执行子系统](./volume-5/index.md)
6. [卷六｜审查协作与高级 runtime](./volume-6/index.md)

---

## 这版入口现在意味着什么

- `guidebookv2/` 是当前正式阅读入口
- 旧的 `00-guidebook/02-15` 已退出主入口，转入 `trash/2026-04-14-guidebookv2-replaced-content/` 保留
- 当前公开阅读应该优先沿六卷问题链进入，而不是回到旧 guidebook 主干顺序

---

## 如果你只想先抓一句 retained takeaway

> **Codex 不是一堆并排功能，而是一套先形成 runtime 主体、再持续存在、再暴露控制面、再管理执行会话、最后继续长成高层运行组织的 agent 系统。**

这就是这版 guidebook v2 要带你读出来的整本书主线。
