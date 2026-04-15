---
title: Codex 卷一写作卡片
date: 2026-04-14
status: draft
tags:
  - Codex
  - guidebook
  - writing-cards
  - volume-1
  - system-overview
  - architecture
---

# Codex 卷一写作卡片

> 用途：作为 Codex guidebook v2 **卷一《系统入口与总图》** 的内部写作卡片。卷一不是先把某条主线讲透，而是先让读者对 Codex 的主要组件、分层关系和全书认知地图建立**初步认识**。后面的卷二到卷六，再分别把这些初步认识补齐、加深、校正。

---

## 卷一总问题

这一卷要回答的是：

> **Codex 到底是什么系统？它由哪些主要组件构成？这些组件分别处在哪一层、扮演什么角色？后面的卷二到卷六，又分别是在补哪一块认识？**

这一卷真正要立住的是：

- Codex 不是 npm CLI、不是单纯 TUI、也不是一坨很厚的 app-server
- 读者要先对 CLI、TUI、app-server、core、ThreadManager、CodexThread、Codex、recovery、unified-exec、skills/MCP、review/guardian/collab/memories 等主要组件有**第一次正确认识**
- 卷一提供的是**整机预览图**，不是局部机制深挖
- 卷二到卷六不是平行材料桶，而是沿着卷一先立起的系统图，逐卷补 runtime core、持续性、控制面、执行子系统与高级 runtime

这卷不是恢复卷，不是控制面卷，不是 unified-exec 卷，也不是平台能力卷。它要做的是：

> **先让读者知道 Codex 这台机器大致长什么样，后面每一卷再带读者走进机器内部。**

---

## 01｜Codex 到底是什么系统：从壳到主体的第一张总图

### 主问题
第一次接触 Codex 时，应该怎样把 CLI、TUI、app-server、core 这些层先分开，知道谁是入口、谁是前端、谁是控制面、谁更接近真正主体？

### 核心判断句
**Codex 是一套以 thread runtime 为核心的 agent 系统：CLI 是入口壳，TUI 是交互前端，app-server 是控制面，真正让系统活起来并持续工作的，是中间那层 runtime core。**

### 这篇必须完成的任务
- 先打掉“Codex = CLI / TUI / 厚 app-server”的常见误读
- 建立卷一的第一张系统总图，让读者先看到分层关系
- 让 `core`、`ThreadManager`、`CodexThread`、`Codex` 这些核心对象第一次出场
- 明确告诉读者：卷一只先建立全景，不在这里把局部机制讲透

### 这篇不讲什么
- 不深讲 thread / turn 语义
- 不深讲 rollout / SQLite / replay 恢复链
- 不深讲 `ServerRequestResolved` / control-plane request semantics
- 不深讲 unified-exec 生命周期
- 不把这篇写成仓库目录导览

### 主图建议
1. `CLI / TUI / app-server / runtime core / execution / capability / advanced runtime` 六层总图
2. `入口壳 -> 前端 -> 控制面 -> runtime core` 的收束图
3. “壳与主体”对照图：哪些是 exposure layer，哪些更接近 owner

### 需要参考的旧文档（绝对路径）
- `~/workspace/codex-source-reading/00-guidebook/01-系统总图与分层.md`
- `~/workspace/knowledge-vault/Inbox/Research/2026-04-12-Codex-卷一-01-总览与架构图-v1.md`
- `~/workspace/knowledge-vault/Inbox/Research/2026-04-12-Codex-卷结构重划与卷一方向校正-记录-v1.md`

### 需要参考的源代码文件（当前本机路径）
- `~/workspace/codex/codex-rs/cli/src/main.rs`
- `~/workspace/codex/codex-rs/tui/src/lib.rs`
- `~/workspace/codex/codex-rs/tui/src/app.rs`
- `~/workspace/codex/codex-rs/app-server/`
- `~/workspace/codex/codex-rs/core/src/lib.rs`
- `~/workspace/codex/codex-rs/core/src/thread_manager.rs`

### 需要重点吸收的 source notes
- `01-source-notes/2026-04-11-codex源码拆解-01-仓库总貌与分层判断.md`
- `03-boundary-judgments/2026-04-11-codex模块边界判断-v1.md`
- 与入口、TUI、app-server、core 分层相关的 call-chain / source-notes

### 对后文的导流
- 第 2 篇进入：不是只知道有哪些层，而是第一次看到一条 interactive 请求怎么穿过这些层
- 第 3 篇进入：每个主要组件先认识一遍，并把后续五卷挂回这张图里

---

## 02｜一条最基础的 interactive 主请求流怎么跑

### 主问题
用户在终端里输入一次请求之后，这个输入是怎么穿过 CLI、TUI、app-server、runtime core，再回到界面的？

### 核心判断句
**卷一不需要把每个内部机制讲透，但必须先让读者看到：Codex 的一次 interactive 请求不是“凭空得到回答”，而是沿着入口、前端、控制面、runtime、执行/采样、事件回流这条正式主链推进的。**

### 这篇必须完成的任务
- 用 preview 级别讲清一次 interactive 请求的最小主链
- 把静态分层图变成动态工作图
- 让读者知道各层在主链上的职责与边界
- 为卷二的 runtime core 主工作回合、卷三的恢复与持续工作线、卷四的控制面投影分别留接口

### 这篇不讲什么
- 不把这篇写成卷二的 runtime core 正文替代品
- 不细讲 turn / pending request / active turn snapshot
- 不细讲 recovery replay
- 不细讲 app-server listener / projection 细节
- 不细讲 unified-exec 内部链

### 主图建议
1. `用户 -> CLI -> TUI -> app-server -> ThreadManager -> CodexThread -> Codex -> action/sampling -> event/result -> app-server -> TUI` 主链图
2. “每一层负责什么 / 不负责什么”的链路标注图
3. “一条请求如何被组织成可继续工作回合”的 preview 图

### 需要参考的旧文档（绝对路径）
- `~/workspace/knowledge-vault/Inbox/Research/2026-04-12-Codex-卷一-01-总览与架构图-v1.md`
- `~/workspace/codex-source-reading/00-guidebook/01-系统总图与分层.md`
- 与 interactive 主链有关的 call-chain 草图

### 需要参考的源代码文件（当前本机路径）
- `~/workspace/codex/codex-rs/cli/src/main.rs`
- `~/workspace/codex/codex-rs/tui/src/app.rs`
- `~/workspace/codex/codex-rs/tui/src/app_server_session.rs`
- `~/workspace/codex/codex-rs/app-server/`
- `~/workspace/codex/codex-rs/core/src/thread_manager.rs`
- `~/workspace/codex/codex-rs/core/src/codex_thread.rs`
- `~/workspace/codex/codex-rs/core/src/codex.rs`

### 需要重点吸收的 source notes
- interactive TUI 主链草图
- app-server / thread / turn 主链相关 source-notes
- 入口、桥接、session 建立相关材料

### 对后文的导流
- 卷二会把这条主链中间的 runtime core 部分真正展开
- 卷三会解释这条链为什么能持续、能恢复
- 卷四会解释这条链怎样被投影成稳定控制面

---

## 03｜Codex 各组件第一次认识：后面每卷分别补哪一块

### 主问题
在还没深入卷二到卷六之前，读者应该先怎样认识 Codex 的主要组件，知道它们各自挂在哪一层、后续去哪一卷补齐？

### 核心判断句
**卷一不是把所有组件讲透，而是先给每个组件一个正确抽屉：先知道它是什么角色、处在哪层、和别的组件怎么分工、后面去哪一卷继续补。**

### 这篇必须完成的任务
- 按“组件群”而不是“源码目录树”组织第一次认识
- 让读者对主要组件建立初步但不错误的定位
- 把卷二到卷六明确挂回每组组件
- 完成卷一最关键的导流功能：让读者知道后面每卷是在补哪块认识

### 建议组件分组
1. **入口与前端**：`codex-cli`、`codex-rs/cli`、TUI
2. **runtime 主体**：`core`、`ThreadManager`、`CodexThread`、`Codex`
3. **状态与持续性**：rollout、SQLite / state、history、recovery / replay
4. **控制面**：app-server、listener、request / event / projection
5. **执行面**：exec、unified-exec、approval、sandbox、transcript、process store
6. **能力与高级 runtime**：skills、MCP、apps / connectors / plugins、review、guardian、collab、memories

### 这篇必须完成的任务细化
- 每组组件都要给出一句定位句
- 每组都要回答“它在系统里不是谁”
- 每组都要给出“后面去哪卷补”的导流句
- 在文末形成一张“卷一预览图 → 卷二到卷六展开图”

### 这篇不讲什么
- 不逐个文件做目录介绍
- 不逐个函数做源码导读
- 不提前进入 control-plane request 细枝末节
- 不提前把 unified-exec、guardian、memories 各自讲成专题

### 主图建议
1. 组件分组总表：组件 / 所在层 / 一句话角色 / 后续卷号
2. 六卷映射图：卷一预览 -> 卷二到卷六分别展开哪块
3. “第一次认识 vs 后面深入认识”的认知坡度图

### 需要参考的旧文档（绝对路径）
- `~/workspace/knowledge-vault/Inbox/Research/2026-04-12-Codex-6卷-index草案-v1.md`
- `~/workspace/knowledge-vault/Inbox/Research/2026-04-12-Codex-卷结构重划与卷一方向校正-记录-v1.md`
- `~/workspace/codex-source-reading/guidebookv2/README.md`
- `~/workspace/codex-source-reading/00-guidebook/00-如何阅读这份导读.md`
- `~/workspace/codex-source-reading/00-guidebook/01-系统总图与分层.md`

### 需要重点吸收的 source notes
- 边界判断材料
- 入口 / 控制面 / 执行面 / review / plugin / MCP / collab / memories 相关的总览型 notes

### 对后文的导流
- 卷二：runtime core 主工作回合怎么跑起来
- 卷三：状态、恢复与持续工作线怎么成立
- 卷四：控制面与 app-server 语义怎么暴露出来
- 卷五：统一执行子系统怎样把动作装成可管理会话
- 卷六：审查协作与高级 runtime 为什么不能被看成边角功能

---

## 卷一的写作红线

### 1. 不把卷一写成目录平扫
卷一服务的是“建立整机认知”，不是“把仓库扫一遍”。

### 2. 不把卷一写成提前透支卷二到卷六
卷一可以 preview，但不能抢后续卷的主工作。

### 3. 每篇都必须带导流
每篇结尾都应该明确告诉读者：下一步去哪一卷补哪块认识。

### 4. 每篇都优先解决读者问题，不优先展示材料堆积
先回答“这是什么 / 它处在哪 / 为什么后面还要分卷”，再谈源码锚点。

---

## 卷一验收标准

读完卷一，读者至少应该能稳定回答：

1. Codex 到底是什么系统，不是什么系统？
2. CLI、TUI、app-server、core 分别是什么层？
3. `ThreadManager`、`CodexThread`、`Codex` 大致分别扮演什么角色？
4. 一条最基础的 interactive 请求大概怎么跑？
5. rollout / recovery / control-plane / unified-exec / review / guardian / collab / memories 各属于哪类问题？
6. 后面卷二到卷六分别是在补哪一块认识？

只要这 6 个问题能答出来，卷一就算真正立住了。