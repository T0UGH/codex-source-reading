---
title: Codex 新卷二写作卡片
date: 2026-04-12
status: draft
tags:
  - Codex
  - guidebook
  - writing-cards
  - volume-2
  - runtime-core
---

# Codex 新卷二写作卡片

> 用途：作为 Codex **新卷二《runtime core 与一条工作回合怎么跑起来》** 的内部写作卡片。先按新卷主问题搭骨架，再从 `00-guidebook/01`、卷一第 01 篇、`02-call-chain-drafts/`、`01-source-notes/` 和相关源码锚点抽素材，禁止让旧卷二的恢复材料反向决定新卷二结构。

---

## 新卷二总问题

这一卷要回答的是：

> **一条请求怎么进入 Codex 的 runtime core，并变成一轮可以持续推进的工作回合？**

这一卷真正要立住的是：

- Codex 的基本运行单位不是一次回复，而是一轮工作回合
- runtime core 不是静态对象集合，而是一条持续推进的主循环
- `ThreadManager` / `CodexThread` / `Codex` 不是平铺对象，而是一条主工作链上的不同层
- action / result / continuation 共同构成 runtime core 的动态闭环

这卷不是恢复卷，也不是控制面卷，更不是 unified-exec 专题卷。它要先把 Codex 的**运行时主线**写稳。

---

## 01｜一次请求怎么进入 Codex 的 runtime 主线

### 主问题
当用户在终端里发起一次交互时，这次输入到底是怎么穿过 CLI、TUI、app-server 这些外层，真正进入 Codex runtime 主线的？

### 核心判断句
**一次请求不是直接落进某个“聊天对象”，而是沿着入口壳、交互前端、控制面桥接，最终进入 runtime core 的正式工作线。**

### 这篇必须完成的任务
- 作为新卷二总起篇，先把“请求进入系统”的主路径写清楚
- 让读者先看到 runtime 不是凭空开始，而是有一个正式接入过程
- 为后文 `ThreadManager` / `CodexThread` / `Codex` 的分层承接铺底

### 这篇不讲什么
- 不细讲恢复与 rollout / SQLite
- 不细讲 app-server 内部 request semantics
- 不细讲 unified-exec 执行链
- 不把这篇写成 CLI 参数导读

### Mermaid 主图
1. `用户 -> CLI -> TUI -> app-server -> ThreadManager -> CodexThread -> Codex` 主链图
2. 入口壳 / 前端 / 控制面 / runtime core 的分层图

### 需要参考的旧文档（绝对路径）
- `~/workspace/codex-source-reading/00-guidebook/01-系统总图与分层.md`

### 需要参考的源代码文件（当前本机路径）
- `~/workspace/codex/codex-rs/cli/src/main.rs`
- `~/workspace/codex/codex-rs/tui/src/lib.rs`
- `~/workspace/codex/codex-rs/tui/src/app.rs`
- `~/workspace/codex/codex-rs/tui/src/app_server_session.rs`
- `~/workspace/codex/codex-rs/core/src/thread_manager.rs`

### 需要重点吸收的 source notes
- 原 03
- 原 04
- 原 interactive TUI 主链草图

### 对后文的导流
- 第 2 篇进入：当前工作回合的 3 个核心对象是怎么接力的
- 第 3 篇进入：当前工作面是怎么被组织出来的

---

## 02｜`ThreadManager`、`CodexThread`、`Codex` 是怎么接成一条 runtime 主工作链的

### 主问题
Codex runtime core 里最关键的三个对象——`ThreadManager`、`CodexThread`、`Codex`——到底各负责什么，它们为什么不是平铺组件，而是一条 runtime 主工作链上的不同层？

### 核心判断句
**`ThreadManager` 管全局 runtime 协调，`CodexThread` 提供线程级交互面，`Codex` 承接更底层的 session/turn loop；三者不是并排摆着的对象，而是一条工作链上的层级接力。**

### 这篇必须完成的任务
- 把三个核心对象的职责切开
- 让读者知道 runtime core 真正的“中间层”由哪些对象承接
- 把卷二从入口路径推进到主对象组织

### 这篇不讲什么
- 不回头重讲 CLI / TUI 外层
- 不提前深讲 turn-history / recovery
- 不深入 app-server 控制面外观
- 不把这篇写成 API 手册

### Mermaid 主图
1. `ThreadManager -> CodexThread -> Codex` 分层接力图
2. 全局协调 / 线程外观 / turn loop 三层职责图

### 需要参考的旧文档（绝对路径）
- `~/workspace/codex-source-reading/00-guidebook/01-系统总图与分层.md`

### 需要参考的源代码文件（当前本机路径）
- `~/workspace/codex/codex-rs/core/src/lib.rs`
- `~/workspace/codex/codex-rs/core/src/thread_manager.rs`
- `~/workspace/codex/codex-rs/core/src/codex_thread.rs`
- `~/workspace/codex/codex-rs/core/src/codex.rs`

### 需要重点吸收的 source notes
- 原 04
- 原 05

### 对后文的导流
- 第 3 篇进入：当前工作面 / current query 是怎么被组织出来的
- 第 4 篇进入：thread 和 turn 在这条工作线里怎么分工

---

## 03｜当前工作面是怎么被组织出来的

### 主问题
一条请求进入 runtime 之后，Codex 不是立刻就“回一句话”，而是会先组织出一个当前工作面；这个工作面到底是什么，它是怎么被组织出来的？

### 核心判断句
**进入 runtime 的输入，不会直接变成回答，而会先被组织成当前这轮工作真正可判断、可继续、可调用能力的工作面。**

### 这篇必须完成的任务
- 把“当前工作面”立成新卷二的核心中间对象
- 解释 runtime 不是拿到输入就输出，而是先组织当前判断上下文
- 为后文“要不要调用能力”“如何继续/收口”建立基础

### 这篇不讲什么
- 不提前写 recovery / replay
- 不提前细讲 control-plane projection
- 不把这篇写成 prompt / context glossary

### Mermaid 主图
1. request -> current working surface / current query 图
2. 输入材料 -> 当前可判断工作面 的组织图

### 需要参考的旧文档（绝对路径）
- `~/workspace/knowledge-vault/Inbox/Research/2026-04-12-Codex-卷一-01-总览与架构图-v1.md`

### 需要参考的源代码文件（当前本机路径）
- `~/workspace/codex/codex-rs/core/src/codex.rs`
- `~/workspace/codex/codex-rs/core/src/codex_thread.rs`
- `~/workspace/codex/codex-rs/app-server/src/codex_message_processor.rs`

### 需要重点吸收的 source notes
- interactive TUI 主链草图
- 与 query / current state 相关的后续 source notes（待补）

### 对后文的导流
- 第 4 篇进入：thread 和 turn 在这里怎么构成工作回合
- 第 5 篇进入：系统怎么决定这一轮要不要调用能力

---

## 04｜thread 和 turn 为什么不是两个平行名词，而是同一条工作线的不同层次

### 主问题
Codex 里 thread 和 turn 为什么不能被理解成两个并排的概念，而应该被理解成同一条工作线上的不同层次？

### 核心判断句
**thread 是持续工作线的承载单元，turn 是这条工作线上的运行轮次；两者不是并列概念，而是容器层与推进层的关系。**

### 这篇必须完成的任务
- 让读者从“对象名词”进入“工作回合结构”
- 把 thread / turn 放回 runtime core 主线，而不是先拖进恢复机制
- 为第 6、7 篇的 action / result / continue-or-stop 铺底

### 这篇不讲什么
- 不展开 turn-history semantic projection（留给新卷三）
- 不展开 active turn snapshot 恢复边界（留给新卷三）
- 不写成纯术语表

### Mermaid 主图
1. thread / turn 的层级图
2. 持续工作线与本轮工作回合的关系图

### 需要参考的旧文档（绝对路径）
- `~/workspace/codex-source-reading/00-guidebook/02-状态持久化与恢复.md`（只取 thread/turn 承载判断，不取恢复展开）

### 需要参考的源代码文件（当前本机路径）
- `~/workspace/codex/codex-rs/core/src/thread_manager.rs`
- `~/workspace/codex/codex-rs/core/src/codex_thread.rs`
- `~/workspace/codex/codex-rs/app-server/src/thread_state.rs`

### 需要重点吸收的 source notes
- 原 17
- 原 43（只取 turn 视图装配意义）

### 对后文的导流
- 第 5 篇进入：系统怎么决定要不要调用能力
- 第 6 篇进入：结果怎么回到当前工作回合

---

## 05｜系统怎么判断这一轮要不要调用能力

### 主问题
进入当前工作面之后，Codex 是怎么从“形成当前判断”推进到“要不要调用能力”的？

### 核心判断句
**Codex 的一轮工作回合不是天然停在首条回复上，而是会先判断当前是该直接收口，还是先进入 action / tool / execution 路径。**

### 这篇必须完成的任务
- 把 runtime core 的决策点写出来
- 解释“回答”与“调用能力”在同一工作回合里如何分叉
- 给卷四执行面和卷五能力面留下自然接口

### 这篇不讲什么
- 不展开具体 unified-exec 内部链
- 不展开 plugins / MCP / skills 全貌
- 不把这篇写成 tool 体系总览

### Mermaid 主图
1. current query -> decide act or answer 图
2. 判断分叉：direct answer vs action path 图

### 需要参考的旧文档（绝对路径）
- 可借鉴 Claude Code 卷一/卷二中“how the system decides to act”的叙述方式，但内容必须以 Codex 为准

### 需要参考的源代码文件（当前本机路径）
- `~/workspace/codex/codex-rs/core/src/codex.rs`
- `~/workspace/codex/codex-rs/core/src/tools/`
- `~/workspace/codex/codex-rs/core/src/unified_exec/`

### 需要重点吸收的 source notes
- 原 exec 链相关草图（只取决策进入执行的边界）
- 与 tool runtime / action path 有关的后续材料

### 对后文的导流
- 第 6 篇进入：动作结果怎么重新回到当前工作回合
- 第 7 篇进入：一轮什么时候继续，什么时候收口

---

## 06｜动作结果怎么重新回到当前工作回合

### 主问题
当系统已经调用了能力、执行了动作之后，结果到底是怎么重新回到当前工作回合里的？

### 核心判断句
**真正关键的不是“执行过一个动作”，而是动作结果必须回到同一轮工作回合，重新成为 runtime 接下来判断的输入。**

### 这篇必须完成的任务
- 把结果回流机制写成新卷二主线的一部分
- 让读者理解 action 不是离开主线，而是短暂出去再回到当前 turn
- 为“继续还是收口”建立直接前提

### 这篇不讲什么
- 不提前深讲 turn-history builder
- 不深讲控制面 notification 语义
- 不把这篇写成 exec plumbing 细节文

### Mermaid 主图
1. action -> result -> continuation 图
2. 当前工作回合里的结果回流闭环图

### 需要参考的旧文档（绝对路径）
- `~/workspace/codex-source-reading/02-call-chain-drafts/2026-04-11-codex主链草图-interactive-TUI链.md`
- `~/workspace/codex-source-reading/02-call-chain-drafts/2026-04-11-codex主链草图-exec链.md`

### 需要参考的源代码文件（当前本机路径）
- `~/workspace/codex/codex-rs/core/src/codex.rs`
- `~/workspace/codex/codex-rs/core/src/tools/handlers/unified_exec.rs`
- `~/workspace/codex/codex-rs/core/src/unified_exec/process_manager.rs`

### 需要重点吸收的 source notes
- 原 exec 链草图
- 原 35 / 38 / 42 / 46 / 49 / 52 / 53 / 55（只取“结果如何回到当前回合”的核心）

### 对后文的导流
- 第 7 篇进入：继续还是收口
- 第 8 篇进入：整条 runtime core 主线稳定运行图

---

## 07｜一轮工作回合什么时候继续，什么时候收口

### 主问题
结果已经回来了，为什么 Codex 有时会继续推进同一轮工作回合，有时会直接收口？这条 continue-or-stop 边界到底在哪？

### 核心判断句
**工作回合的边界不在“做没做动作”，而在 runtime 判断当前结果是否已经足够支持收口；只要还不够，这一轮通常就会继续。**

### 这篇必须完成的任务
- 把 continue-or-stop 边界立成新卷二卷尾前的关键判断
- 让读者理解一轮工作回合为什么不是一问一答
- 为最终稳定运行图收口

### 这篇不讲什么
- 不回头重讲恢复机制
- 不扩展到 app-server request replay
- 不把这篇写成产品交互规则表

### Mermaid 主图
1. result -> continue or stop 图
2. 一轮工作回合的收口边界图

### 需要参考的旧文档（绝对路径）
- 可借鉴 Claude Code 卷二 07 的问题意识，但内容要以 Codex runtime 为准

### 需要参考的源代码文件（当前本机路径）
- `~/workspace/codex/codex-rs/core/src/codex.rs`
- `~/workspace/codex/codex-rs/core/src/codex_thread.rs`

### 需要重点吸收的 source notes
- 与 turn 终态 / next step / current event 相关的后续材料

### 对后文的导流
- 第 8 篇进入：整条 runtime core 稳定运行图
- 新卷三进入：这条工作回合为什么能被保存、恢复、继续

---

## 08｜把整条 runtime core 主线重新压成一张稳定运行图

### 主问题
如果把前面几篇全部收回来，Codex 的 runtime core 到底应该在读者脑中留下怎样一张稳定运行图？

### 核心判断句
**Codex 的核心不是一组静态模块，而是一条会持续推进的工作回合主线：请求进入、工作面形成、当前判断产生、必要时进入动作、结果回流、决定继续或收口。**

### 这篇必须完成的任务
- 作为新卷二卷尾，把整卷重新压成一张稳定运行图
- 让读者带着“runtime core 动态主线”进入新卷三、新卷四
- 明确把恢复和控制面都挂回 runtime core

### 这篇不讲什么
- 不重新展开全部细节
- 不把这篇写成卷一 preview 的重复版
- 不把恢复和控制面写穿

### Mermaid 主图
1. Codex runtime core 稳定运行总图
2. runtime core 与卷三（恢复）、卷四（控制面）、卷五（执行/能力）的挂接图

### 需要参考的旧文档（绝对路径）
- `~/workspace/knowledge-vault/Inbox/Research/2026-04-12-Codex-卷一-01-总览与架构图-v1.md`
- `~/workspace/codex-source-reading/02-call-chain-drafts/2026-04-11-codex主链草图-interactive-TUI链.md`

### 需要参考的源代码文件（当前本机路径）
- 以卷内前文已经锚定过的 runtime core 主链为准

### 对后文的导流
- 新卷三进入：为什么这条工作回合能被保存、恢复、继续
- 新卷四进入：这条工作回合怎样被暴露成控制面

---

## 当前阶段结论

新卷二第一版先收成 8 篇，理由是：

1. 足够把 runtime core 单独立成一卷
2. 足够把“工作回合”写成一条动态主线
3. 不会再让恢复机制过早顶替 runtime 主线
4. 能和新卷三、新卷四形成稳定边界

## 下一步建议

如果这版方向成立，下一步最值的动作顺序应该是：

1. 起 **新卷二 README-production-order.md**
2. 做 **旧材料 / 现有文章 到新卷二的迁移表**
3. 再开始写新卷二正文
