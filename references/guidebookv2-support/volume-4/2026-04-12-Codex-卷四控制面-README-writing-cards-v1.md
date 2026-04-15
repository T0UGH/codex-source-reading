---
title: Codex 卷四写作卡片
date: 2026-04-12
status: draft
tags:
  - Codex
  - guidebook
  - writing-cards
  - volume-4
---

# Codex 卷四写作卡片

> 用途：作为 Codex 新卷四《Codex 的控制面怎么把 runtime 暴露出来》的内部写作卡片。先按新卷主问题搭骨架，再从 `00-guidebook/03`、`14`、`03-boundary-judgments/`、`02-call-chain-drafts/` 和 `01-source-notes/` 抽素材，禁止让旧正文反向决定新文章结构。

---

## 卷四总问题

这一卷要回答的是：

> **thread runtime 已经成立之后，Codex 怎样把它稳定投影成 TUI、SDK、远端调用者可观察、可控制、可恢复连接的控制面？**

这一卷不是 app-server 文件目录导读，也不是 request type 清单。它真正要立住的是：

- app-server 为什么不是另一套 runtime
- listener / projection / state correction 怎样把控制面撑起来
- request semantics 为什么不是一层意思
- TUI 为什么越来越像跑在 app-server 之上

---

## 01｜为什么 app-server 不是另一套 runtime，而是 control-plane facade

### 主问题
为什么看起来很厚的 app-server，不该被理解成 Codex 的另一套平行 runtime，而应该被理解成建立在 core runtime 之上的控制面 facade？

### 核心判断句
**app-server 的职责不是重新持有 thread runtime，而是把 core 里已经成立的 live thread / session 语义投影成可调用、可观察、可恢复连接的稳定控制面。**

### 这篇必须完成的任务
- 作为卷四总起篇，先把 runtime owner 和 control-plane facade 分开
- 解决“Codex 看起来像 app-server 产品”带来的第一层误读
- 让读者理解卷四不是讲“服务端目录”，而是讲“控制面如何成立”

### 这篇不讲什么
- 不深入 listener 的细节执行链
- 不展开 `ServerRequestResolved` 的分类问题
- 不提前进入 unified-exec
- 不把这篇写成 app-server API 概览

### Mermaid 主图
1. `core / ThreadManager / app-server / TUI` 的控制面分层图
2. runtime owner 与 facade 的边界图

### 需要参考的旧文档（绝对路径）
- `~/workspace/codex-source-reading/00-guidebook/03-app-server与thread-turn主线.md`
- `~/workspace/codex-source-reading/00-guidebook/01-系统总图与分层.md`
- `~/workspace/codex-source-reading/03-boundary-judgments/2026-04-11-codex模块边界判断-v1.md`

### 需要参考的源代码文件（当前本机路径）
- `~/workspace/codex/codex-rs/app-server/src/lib.rs`
- `~/workspace/codex/codex-rs/app-server/src/message_processor.rs`
- `~/workspace/codex/codex-rs/core/src/thread_manager.rs`
- `~/workspace/codex/codex-rs/tui/src/app_server_session.rs`

### 对后文的导流
- 第 2 篇进入 listener / projection 主链
- 第 3 篇进入 request semantics

---

## 02｜listener task、协议投影与状态修正是怎么把控制面撑起来的

### 主问题
Codex 的控制面不是靠一个“大服务对象”直接撑住的，那 listener task、`bespoke_event_handling`、状态修正函数到底是怎么一起把 thread runtime 投影成稳定控制面的？

### 核心判断句
**Codex 的控制面不是一个单点大状态机，而是 listener task 作为线程事件泵、`bespoke_event_handling` 作为协议投影层，再加上一组状态修正与收口函数，共同把 runtime 事件整理成稳定控制面。**

### 这篇必须完成的任务
- 把 listener task 立成 thread-event pump
- 把 `bespoke_event_handling` 立成协议投影层
- 解释为什么 `thread_state_manager` / `thread_watch_manager` 是有意拆分
- 把卷四的“控制面成立条件”真正写厚

### 这篇不讲什么
- 不深讲 turn-history 语义权威问题（那是卷二）
- 不深讲 request resolved 的分类表
- 不讲 unified-exec 执行链
- 不把状态修正函数写成逐函数百科

### Mermaid 主图
1. `listener -> projection -> outward status` 控制面主链图
2. `thread_state_manager / thread_watch_manager` 分工图

### 需要参考的旧文档（绝对路径）
- `~/workspace/codex-source-reading/00-guidebook/03-app-server与thread-turn主线.md`
- `~/workspace/codex-source-reading/02-call-chain-drafts/2026-04-11-codex主链草图-interactive-TUI链.md`

### 需要参考的源代码文件（当前本机路径）
- `~/workspace/codex/codex-rs/app-server/src/codex_message_processor.rs`
- `~/workspace/codex/codex-rs/app-server/src/bespoke_event_handling.rs`
- `~/workspace/codex/codex-rs/app-server/src/thread_state.rs`
- `~/workspace/codex/codex-rs/app-server/src/outgoing_message.rs`

### 需要重点吸收的 source notes
- 原 32 / 33 / 36 / 37 / 39 / 41 / 44 / 45 / 48

### 对后文的导流
- 第 3 篇进入 `ServerRequestResolved` 和 request shape 分类
- 第 4 篇进入 `DynamicToolCall` 边界

---

## 03｜`ServerRequestResolved` 到底覆盖了什么控制面语义

### 主问题
`ServerRequestResolved` 在 Codex 控制面里到底覆盖了哪些 request 语义，为什么它不能被简单理解成“所有 request 最后都会走的统一 resolved 事件”？

### 核心判断句
**`ServerRequestResolved` 不是 request 系统的总括名，而是 Codex 在一批 V2 thread-scoped interactive request 上形成的 resolved-notification 语义模型。**

### 这篇必须完成的任务
- 把 callback-map resolution 和 `ServerRequestResolved` 分开
- 解释这条 resolved 语义模型当前到底覆盖哪些 request
- 让读者能看懂卷四后半的 request semantics 不再是一团

### 这篇不讲什么
- 不重新回到 app-server 总体结构
- 不详细展开 `DynamicToolCall` 的 item lifecycle（留给下一篇）
- 不把 deprecated V1 holdouts 写成这篇主线
- 不把全文写成 enum 表注释

### Mermaid 主图
1. request transport / callback resolution / resolved semantics 三层边界图
2. 当前 request shape 分类图（resolved / semantic split / global bridge / legacy holdouts）

### 需要参考的旧文档（绝对路径）
- `~/workspace/codex-source-reading/00-guidebook/14-ServerRequestResolved覆盖面与未迁移疑点.md`
- `~/workspace/codex-source-reading/03-boundary-judgments/2026-04-12-app-server-request-shape分类与收口.md`

### 需要参考的源代码文件（当前本机路径）
- `~/workspace/codex/codex-rs/app-server/src/message_processor.rs`
- `~/workspace/codex/codex-rs/app-server/src/models.rs`
- `~/workspace/codex/codex-rs/app-server-protocol/src/lib.rs`

### 需要重点吸收的 source notes
- 原 40
- 以及与 pending request / state correction 有关的原 39 / 44 / 48

### 对后文的导流
- 第 4 篇专门切到 `DynamicToolCall` 为什么不走 resolved 语义
- 第 5 篇收口 TUI over app-server

---

## 04｜为什么 `DynamicToolCall` 不走 `ServerRequestResolved`，而走 item lifecycle

### 主问题
既然 `DynamicToolCall` 看起来也是一类 thread-scoped request，为什么它不进入 `ServerRequestResolved` 语义模型，而要通过 item lifecycle 处理？

### 核心判断句
**`DynamicToolCall` 并不是“还没迁移进 resolved-notification 模型的洞”，而是 Codex 在 transport 上复用 request machinery、在产品语义上转向 item lifecycle 的一条有意分叉。**

### 这篇必须完成的任务
- 把 `DynamicToolCall` 从“可疑缺口”重新压成“语义分叉点”
- 让读者理解 transport reuse 和 product semantics split 不是一回事
- 稳住卷四后半最容易反复误判的边界

### 这篇不讲什么
- 不重新解释所有 request type
- 不扩展到 unified-exec 或 plugin 系统
- 不把这篇写成 open question 日志

### Mermaid 主图
1. `DynamicToolCall`：transport reuse vs item lifecycle semantics 图
2. `ServerRequestResolved` 与 `ItemStarted/ItemCompleted` 的分叉对照图

### 需要参考的旧文档（绝对路径）
- `~/workspace/codex-source-reading/03-boundary-judgments/2026-04-12-DynamicToolCall为什么不走ServerRequestResolved.md`
- `~/workspace/codex-source-reading/03-boundary-judgments/2026-04-12-app-server-request-shape分类与收口.md`

### 需要参考的源代码文件（当前本机路径）
- `~/workspace/codex/codex-rs/app-server/src/message_processor.rs`
- `~/workspace/codex/codex-rs/app-server/src/codex_message_processor.rs`
- `~/workspace/codex/codex-rs/app-server-protocol/src/lib.rs`

### 对后文的导流
- 第 5 篇从 request semantics 收口到 TUI over app-server 的整体控制面接口

---

## 05｜为什么 TUI 越来越像跑在 app-server 之上，而不是直接抓 core

### 主问题
如果 runtime 在 core、控制面在 app-server，那今天的 Codex 体验为什么越来越表现为 TUI over app-server，而不是 TUI 直接抓 core？

### 核心判断句
**TUI 越来越像跑在 app-server 之上，不是因为 app-server 比 core 更底层，而是因为 app-server 更适合作为统一控制面 contract，把 embedded 和 remote 两种形态收成同一交互入口。**

### 这篇必须完成的任务
- 把卷四前面几篇的判断收成一个读者可记住的控制面结论
- 解释 embedded app-server / remote app-server / `AppServerSession` 的统一接口意义
- 作为卷四收口篇，把“控制面”从模块知识压成系统判断

### 这篇不讲什么
- 不进入 unified-exec 会话执行链
- 不扩展到平台层和协作层
- 不把这篇写成 TUI UI 功能导览

### Mermaid 主图
1. `core -> app-server -> AppServerSession -> TUI` 统一接口图
2. embedded vs remote app-server 的 transport variation 图

### 需要参考的旧文档（绝对路径）
- `~/workspace/codex-source-reading/00-guidebook/01-系统总图与分层.md`
- `~/workspace/codex-source-reading/00-guidebook/03-app-server与thread-turn主线.md`
- `~/workspace/codex-source-reading/01-source-notes/2026-04-11-codex源码拆解-20-TUI向app-server收敛的现状.md`
- `~/workspace/codex-source-reading/01-source-notes/2026-04-11-codex源码拆解-23-remote-app-server与websocket链路.md`

### 需要参考的源代码文件（当前本机路径）
- `~/workspace/codex/codex-rs/tui/src/app_server_session.rs`
- `~/workspace/codex/codex-rs/tui/src/app.rs`
- `~/workspace/codex/codex-rs/app-server-client/src/lib.rs`
- `~/workspace/codex/codex-rs/app-server-client/src/remote.rs`

### 对后文的导流
- 卷四进入真正执行会话主线：unified-exec
- 卷四在这里收住为：Codex 已经长出正式控制面，而不是仅靠 UI 直接驱动 runtime

---

## 当前阶段结论

卷四卡片第一版先收成 5 篇，理由是：

1. 已经足够把控制面立成一个完整问题链
2. 不会一上来把 request semantics 拆得过细
3. 可以和卷四形成自然导流
4. 最适合先从现有 `00-guidebook/03`、`14` 和 boundary docs 抽素材

## 下一步建议

如果这一版方向成立，下一步不是先写第 1 篇正文，而是先补：

1. 卷四 `README-production-order.md`
2. 再起每篇的篇级材料清单 / 源码锚点清单
3. 然后再开始跑正文
