---
title: Codex 卷五写作卡片
date: 2026-04-13
status: draft
tags:
  - Codex
  - guidebook
  - writing-cards
  - volume-5
  - unified-exec
---

# Codex 卷五写作卡片

> 用途：作为 Codex 新卷五《统一执行子系统怎样把动作变成可管理会话》的内部写作卡片。先按卷级主问题搭骨架，再从 `00-guidebook/05`、`02-call-chain-drafts/2026-04-11-codex主链草图-exec链.md`、相关 `01-source-notes/` 和源码锚点抽素材，禁止让旧正文的章节顺序直接决定新卷五结构。

---

## 卷五总问题

这一卷要回答的是：

> **当 Codex 真正进入动作执行时，为什么它不是一个 shell wrapper，而是一套 sessionful、approval-aware、streamable、可对账的统一执行子系统？**

这一卷真正要立住的是：

- `exec.rs` 和 unified-exec 不是一回事
- `UnifiedExecHandler` / `UnifiedExecRuntime` 不是薄转发，而是在把动作装成正式执行会话
- approval / sandbox / policy 不是执行外围，而是在执行前就进入主链
- 输出不是“跑完再拼字符串”，而是先进入 transcript / delta 流
- process store 不是最终语义真相，而是执行会话生命周期的存储与对账层

一句话压缩：

> **卷五负责把 Codex 作为一套“会把动作组织成可管理会话”的系统讲明白。**

这卷不是 shell 命令教程，也不是环境配置说明，更不是 review/guardian 专题。

---

## 建议篇章方向（v1）

### 01｜为什么 `exec.rs` 和 unified-exec 不是一回事

#### 主问题
很多人第一次看到执行层，会把 unified-exec 理解成“比 `exec.rs` 稍厚一点的执行封装”。为什么这个理解太弱了？

#### 核心判断句
**`exec.rs` 更像执行 primitive 层；unified-exec 则是在其上长出了一层 agent-facing、sessionful、approval-aware 的执行会话子系统。**

#### 这篇必须完成的任务
- 把 primitive exec 和 unified-exec subsystem 的层级切开
- 让读者先知道卷五讲的不是“怎么跑命令”，而是“怎么把执行变成一条正式运行链”
- 给后文的 handler / runtime / transcript / process store 铺底

#### 这篇不讲什么
- 不提前深讲 approval 分支细节
- 不展开 transcript 输出链细节
- 不展开 exec-server / sandbox 后端全貌

#### Mermaid 主图
1. `exec.rs -> unified_exec` 的分层图
2. primitive execution vs managed execution session 对照图

#### 需要参考的旧文档（绝对路径）
- `~/workspace/codex-source-reading/00-guidebook/05-unified-exec执行子系统.md`

#### 需要参考的 source notes
- 原 12
- 原 35

#### 对后文导流
- 第 02 篇进入：执行请求怎样被装配成正式会话
- 第 03 篇进入：approval / sandbox / policy 怎样在执行前进入主链

---

### 02｜`UnifiedExecHandler` 和 `UnifiedExecRuntime` 是怎么把动作装成执行会话的

#### 主问题
Codex 在真正执行动作时，为什么不是“收到 tool call 就直接 spawn”，而是先经过 handler 和 runtime 这两层装配？

#### 核心判断句
**`UnifiedExecHandler` 负责把动作请求装成带身份的执行会话入口，`UnifiedExecRuntime` 负责把 approval / policy / environment / runtime route 纳入正式主链；两者共同决定 unified-exec 是子系统，而不是薄封装。**

#### 这篇必须完成的任务
- 切开 handler 与 runtime 的职责
- 说明 process/session 身份是在这里被正式建立的
- 让读者看到 unified-exec 的入口并不等于“直接跑命令”

#### 这篇不讲什么
- 不深入输出流处理细节
- 不提前讲最终 end event 封装
- 不把全文写成函数逐段解释

#### Mermaid 主图
1. action request -> handler -> runtime -> open session 主链图
2. request assembly / runtime route / session identity 三层图

#### 需要参考的旧文档（绝对路径）
- `~/workspace/codex-source-reading/00-guidebook/05-unified-exec执行子系统.md`

#### 需要参考的 source notes
- 原 35
- 原 38
- 原 42

#### 对后文导流
- 第 03 篇进入：approval / sandbox / policy 为什么在执行前就进入主链
- 第 04 篇进入：输出为什么先走 transcript

---

### 03｜为什么 approval、sandbox、policy 不是执行外围，而是在执行前就进入主链

#### 主问题
为什么在 Codex 里，approval、sandbox、network/policy 判断，不是“命令跑起来以后再处理的外围逻辑”，而是在真正执行前就进入统一执行主链？

#### 核心判断句
**unified-exec 的关键不在“先执行、再补安全”，而在“执行前就把 approval / sandbox / policy / runtime route 纳入统一控制链”。**

#### 这篇必须完成的任务
- 把 approval-aware / policy-aware 写成 unified-exec 的本体能力，而不是附属检查
- 解释为什么这一步决定了 Codex 执行层不是 shell wrapper
- 为后文的 transcript / end event / process store 铺垫执行会话心智

#### 这篇不讲什么
- 不展开 guardian/review 专题
- 不深讲每一种 sandbox backend
- 不把这篇写成权限系统总览

#### Mermaid 主图
1. request -> approval/policy -> runtime route -> execution session 图
2. “先执行再处理” vs “执行前进入主链” 对照图

#### 需要参考的旧文档（绝对路径）
- `~/workspace/codex-source-reading/00-guidebook/05-unified-exec执行子系统.md`

#### 需要参考的 source notes
- 原 08
- 原 27
- 原 38
- 原 42

#### 对后文导流
- 第 04 篇进入：输出为什么先进入 transcript / delta
- 第 05 篇进入：为什么 process store 不是最终权威源

---

### 04｜输出为什么先进入 transcript，而不是直接变成“最终结果”

#### 主问题
为什么 unified-exec 的输出不是“命令结束后把 stdout/stderr 拼起来就完了”，而是会先进入 transcript、delta 流和聚合输出链？

#### 核心判断句
**在 unified-exec 里，输出首先是一条正在进行中的事件流，先进入 transcript / delta 才能成立为可流式观察、可统一收尾、可继续被 runtime 理解的执行会话材料。**

#### 这篇必须完成的任务
- 把 transcript 立成主语义轨，而不是附属日志
- 解释 `process_chunk` / UTF-8 边界 / aggregated output 这些小函数为什么关键
- 让读者理解流式输出与最终结果并不是同一层东西

#### 这篇不讲什么
- 不深讲 process store 对账
- 不把这篇写成 stdout/stderr 工程细节文
- 不提前承担卷尾总结职责

#### Mermaid 主图
1. process output -> chunk -> transcript delta -> aggregated output 图
2. raw stream / transcript / final output 三层图

#### 需要参考的旧文档（绝对路径）
- `~/workspace/codex-source-reading/00-guidebook/05-unified-exec执行子系统.md`

#### 需要参考的 source notes
- 原 49
- 原 52
- 原 55
- 原 56

#### 对后文导流
- 第 05 篇进入：process store 为什么不是最终真相，而是会话生命周期的存储与对账层

---

### 05｜为什么 process store 不是最终权威源，而 unified-exec 更像 execution control plane

#### 主问题
如果 unified-exec 已经有 process store，为什么它仍不能被理解成执行结果的最终真相源？为什么更准确的理解，是它在把执行会话组织成一套 execution control plane？

#### 核心判断句
**process store 记录的是进程生命周期与状态对账，不是最终语义权威；真正值钱的是 unified-exec 把动作装成了一个可批准、可流式观察、可统一封装终态、可持续对账的 execution control plane。**

#### 这篇必须完成的任务
- 把 transcript / process store / end event 三者关系压成稳定结论
- 让卷五最后收束到“管理执行会话”，而不是“管理命令调用”
- 作为卷尾，把 unified-exec 立成正式 execution control plane

#### 这篇不讲什么
- 不扩展到 plugin/MCP 平台层
- 不把全文写成 process registry 细节索引
- 不提前进入下一卷的 capability system

#### Mermaid 主图
1. execution session lifecycle：request -> session -> stream -> end -> reconcile 图
2. transcript / process store / end event 的职责边界图

#### 需要参考的旧文档（绝对路径）
- `~/workspace/codex-source-reading/00-guidebook/05-unified-exec执行子系统.md`

#### 需要参考的 source notes
- 原 46
- 原 53
- 原 54
- 原 57

#### 对后文导流
- 下一卷进入：能力系统怎样建立在执行面之上，把 Codex 长成平台

---

## 这一卷的固定主线顺序

> **primitive exec 边界 → 会话装配入口 → 执行前控制链 → transcript 输出语义 → process store / end event / execution control plane 收口**

这条顺序不能随便改。原因很简单：

- 如果太早讲 transcript，读者会不知道它为什么不是普通日志
- 如果太早讲 process store，读者会把整卷读成进程管理专题
- 如果太早讲 sandbox/policy，全卷会滑成权限与环境专题
- 只有先立住“这不是普通 exec，而是执行会话子系统”，后面几篇才会自然成立

---

## 当前默认边界

### 这卷不应该吃掉什么
- 不吃卷四 control plane 的 request semantics 与 app-server 投影主线
- 不吃下一卷 capability system 的 MCP / skills / plugins 平台化问题
- 不把 guardian / review / advanced runtime 拉进来

### 这卷最该留下来的记忆点

> **Codex 的动作执行，不是“一次命令调用”，而是一套被统一装配、可被批准、可被流式观察、可统一收尾、可持续对账的执行会话系统。**
