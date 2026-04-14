---
title: Codex 卷三写作卡片
date: 2026-04-12
status: draft
tags:
  - Codex
  - guidebook
  - writing-cards
  - volume-3
---

# Codex 卷三写作卡片

> 用途：作为 Codex 新卷三《线程、状态与恢复是怎么成立的》的内部写作卡片。先按新卷主问题搭骨架，再从 `00-guidebook/02`、`00-guidebook/04`、`01-source-notes/` 和相关源码锚点抽素材，禁止让旧正文反向决定新文章结构。

---

## 卷三总问题

这一卷要回答的是：

> **Codex 为什么不是“一次命令执行就结束”，而是一个可以持续、恢复、继续运行的线程化系统？**

这一卷真正要立住的是：

- rollout 和 SQLite 不是什么关系
- 恢复为什么更像 replay，而不是查库拼对象
- thread / turn / pending request 怎样组成持续工作线
- turn-history 为什么不是 event log 镜像，而是一层 semantic projection

卷三不是数据库导读，也不是 turn 名词表，而是要把 Codex 的**持续工作条件**写稳。

---

## 01｜为什么 rollout 才是正文真相源，而 SQLite 不是恢复核心

### 主问题
为什么 Codex 在状态与恢复问题上，真正更像依赖 rollout 这类 durable history truth source，而不是依赖 SQLite 直接把会话对象查回来？

### 核心判断句
**Codex 的恢复真相主要不在 SQLite，而在 rollout 这类可 replay 的正文历史；SQLite 更像 metadata / index / sidecar state，而不是恢复主脑。**

### 这篇必须完成的任务
- 作为卷三总起篇，先把 rollout 与 SQLite 的职责分开
- 解决“有数据库就等于数据库负责恢复”的直觉误读
- 为后面 recovery / turn-history 的 replay 心智铺底

### 这篇不讲什么
- 不深入 turn-history builder 细节
- 不提前展开 app-server 控制面投影
- 不进入 unified-exec 执行子系统
- 不把这篇写成存储文件清单

### Mermaid 主图
1. rollout / SQLite / config / state sidecar 的职责分层图
2. 正文真相源 vs metadata sidecar 的边界图

### 需要参考的旧文档（绝对路径）
- `~/workspace/codex-source-reading/00-guidebook/02-状态持久化与恢复.md`
- `~/workspace/codex-source-reading/00-guidebook/01-系统总图与分层.md`

### 需要参考的源代码文件（当前本机路径）
- `~/workspace/codex/codex-rs/core/src/rollout.rs`
- `~/workspace/codex/codex-rs/rollout/src/lib.rs`
- `~/workspace/codex/codex-rs/core/src/state_db_bridge.rs`
- `~/workspace/codex/codex-rs/state/src/lib.rs`
- `~/workspace/codex/codex-rs/core/src/state/service.rs`

### 需要重点吸收的 source notes
- 原 06
- 原 11
- 原 18

### 对后文的导流
- 第 2 篇进入 recovery 为什么是 replay
- 第 3 篇进入 thread / turn / pending request 的持续工作线

---

## 02｜为什么恢复更像 replay，而不是“查库拼对象”

### 主问题
Codex 的恢复为什么不该被理解成数据库查询 + 对象重建，而更应该被理解成一条 replay / reconstruction 主线？

### 核心判断句
**Codex 的恢复不是 ORM 式查表重建，而是把持久化历史重新 replay 成还能继续工作的当前语义状态。**

### 这篇必须完成的任务
- 把 recovery 从“查库”心智切换成“replay”心智
- 解释 reconstruction 处理的不是简单数据回填，而是 surviving history 的语义重建
- 为后面 turn-history 不是 event log 镜像这一点继续铺垫

### 这篇不讲什么
- 不详细展开 `ThreadHistoryBuilder` 所有分支
- 不展开 app-server request semantics
- 不进入 live control-plane 投影细节

### Mermaid 主图
1. persistence -> replay -> reconstruction -> current workable state 图
2. “查库拼对象” vs “replay 语义重建” 对照图

### 需要参考的旧文档（绝对路径）
- `~/workspace/codex-source-reading/00-guidebook/02-状态持久化与恢复.md`
- `~/workspace/codex-source-reading/00-guidebook/04-turn-history语义层.md`

### 需要参考的源代码文件（当前本机路径）
- `~/workspace/codex/codex-rs/core/src/codex/rollout_reconstruction.rs`
- `~/workspace/codex/codex-rs/core/src/codex.rs`
- `~/workspace/codex/codex-rs/core/src/thread_rollout_truncation.rs`
- `~/workspace/codex/codex-rs/rollout/src/lib.rs`

### 需要重点吸收的 source notes
- 原 34
- 原 11
- 原 18

### 对后文的导流
- 第 3 篇进入 thread / turn / pending request 怎样成为持续工作单元
- 第 4 篇进入 turn-history semantic projection

---

## 03｜thread、turn 与 pending request 是怎么一起组成持续工作线的

### 主问题
Codex 为什么不是“保存一堆消息 + 下次再读回来”，而是通过 thread、turn、pending request 这几层对象，把会话真正组织成一条可持续、可恢复、可继续推进的工作线？

### 核心判断句
**Codex 的持续工作能力，不是靠“多存一点历史”成立的，而是靠 thread 作为承载单元、turn 作为运行轮次、pending request 作为中途未决状态，共同组织出一条可以继续往前跑的工作线。**

### 这篇必须完成的任务
- 把 thread 立成真正的会话承载单元
- 把 turn 立成 runtime 轮次，而不是简单文本轮次
- 把 pending request 写成持续工作线的一部分，而不是异常附属状态
- 把卷三从“存储问题”推进到“运行状态问题”

### 这篇不讲什么
- 不展开 listener / projection / app-server 控制面（留给卷三）
- 不细讲 turn-history builder 的所有归约规则
- 不进入 unified-exec 生命周期

### Mermaid 主图
1. thread / turn / pending request 的持续工作线分层图
2. “历史材料”与“当前可继续运行状态”的区别图

### 需要参考的旧文档（绝对路径）
- `~/workspace/codex-source-reading/00-guidebook/02-状态持久化与恢复.md`
- `~/workspace/codex-source-reading/00-guidebook/04-turn-history语义层.md`

### 需要参考的源代码文件（当前本机路径）
- `~/workspace/codex/codex-rs/core/src/thread_manager.rs`
- `~/workspace/codex/codex-rs/core/src/codex_thread.rs`
- `~/workspace/codex/codex-rs/app-server/src/thread_state.rs`
- `~/workspace/codex/codex-rs/app-server/src/codex_message_processor.rs`

### 需要重点吸收的 source notes
- 原 17
- 原 43
- 原 58

### 对后文的导流
- 第 4 篇进入 turn-history 为什么不是 event log 镜像
- 第 5 篇进入 active turn / user message / current projection 的边界

---

## 04｜为什么 turn-history 不是 event log 镜像，而是一层 semantic projection

### 主问题
Codex 的 turn-history 到底是什么对象，为什么它不是把 event stream 原样堆出来，而是要通过 builder / replay / reconstruction 重新组织成一层 turn semantic projection？

### 核心判断句
**turn-history 不是 event log 的镜像，而是 Codex 把 live / persisted events 重新归约成可展示、可恢复、可继续推进的 turn 语义视图。**

### 这篇必须完成的任务
- 把 `ThreadHistoryBuilder` 立成 turn 语义归约器
- 解释显式 turn 生命周期和旧流隐式边界如何并存
- 让读者理解“按到达顺序堆历史”不是这套系统的正确心智

### 这篇不讲什么
- 不把 reconstruction 与 builder 混成一层
- 不深入 app-server 协议面细节
- 不把这篇写成 event type 逐条注释

### Mermaid 主图
1. event stream -> ThreadHistoryBuilder -> turn history 的归约图
2. arrival order vs semantic association 的对照图

### 需要参考的旧文档（绝对路径）
- `~/workspace/codex-source-reading/00-guidebook/04-turn-history语义层.md`

### 需要参考的源代码文件（当前本机路径）
- `~/workspace/codex/codex-rs/app-server-protocol/src/protocol/thread_history.rs`
- `~/workspace/codex/codex-rs/core/src/codex.rs`
- `~/workspace/codex/codex-rs/core/src/codex/rollout_reconstruction.rs`
- `~/workspace/codex/codex-rs/app-server/src/thread_state.rs`

### 需要重点吸收的 source notes
- 原 47
- 原 50
- 原 34

### 对后文的导流
- 第 5 篇进入 `active_turn_snapshot` / `handle_user_message` 的当前投影边界

---

## 05｜`active_turn_snapshot`、`handle_user_message` 与当前工作视图的边界

### 主问题
在 turn-history 已经成立之后，Codex 怎样决定“当前/最近哪个 turn 可以拿来对外展示或继续工作”，以及为什么 `active_turn_snapshot` 和 `handle_user_message` 不是普通 getter / append 函数，而是当前工作视图的边界函数？

### 核心判断句
**`active_turn_snapshot` 管的不是“取当前 turn”这么简单，`handle_user_message` 管的也不是“记一条用户输入”这么简单；它们共同决定的是当前工作视图从哪开始、当前 turn 怎样收口、新 turn 怎样开出来。**

### 这篇必须完成的任务
- 把 `active_turn_snapshot` 写成当前/最近 turn 的对外投影口
- 把 `handle_user_message` 写成 implicit turn 边界函数
- 作为卷三收口篇，把“持续工作”真正收成一条可恢复、可续接、可投影的工作线

### 这篇不讲什么
- 不重新展开 persistence 分层
- 不进入卷三的 control-plane facade 话题
- 不进入 request semantics / resolved 模型

### Mermaid 主图
1. user message / active turn / current projection 的边界图
2. persisted history / active turn / outward snapshot 的关系图

### 需要参考的旧文档（绝对路径）
- `~/workspace/codex-source-reading/00-guidebook/04-turn-history语义层.md`
- `~/workspace/codex-source-reading/00-guidebook/02-状态持久化与恢复.md`

### 需要参考的源代码文件（当前本机路径）
- `~/workspace/codex/codex-rs/core/src/codex.rs`
- `~/workspace/codex/codex-rs/app-server/src/thread_state.rs`
- `~/workspace/codex/codex-rs/app-server/src/codex_message_processor.rs`

### 需要重点吸收的 source notes
- 原 51
- 原 58
- 辅助参考原 43 / 47 / 50

### 对后文的导流
- 卷三进入：Codex 的控制面怎样把这条已经成立的 runtime 工作线暴露出来
- 卷三在这里收住为：Codex 的持续工作条件已经成立，不只是“存了历史”，而是 runtime 能重建一条还能继续跑的工作线

---

## 当前阶段结论

卷三卡片第一版先收成 5 篇，理由是：

1. 已经足够把 persistence / replay / thread-turn / semantic projection 讲成一条连续问题链
2. 不会一上来就把 turn-history 和 app-server 控制面混在一起
3. 可以和卷三形成自然导流
4. 最适合先从 `00-guidebook/02`、`00-guidebook/04` 和原 06 / 11 / 17 / 18 / 34 / 43 / 47 / 50 / 51 / 58 抽素材

## 下一步建议

如果这一版方向成立，下一步不是直接写第 1 篇正文，而是先补：

1. 卷三 `README-production-order.md`
2. 再起每篇的篇级材料清单 / 源码锚点清单
3. 然后再开始跑正文
