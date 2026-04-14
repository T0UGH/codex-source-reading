---
title: Codex 卷五执行说明
date: 2026-04-13
status: draft
tags:
  - Codex
  - guidebook
  - production-order
  - volume-5
  - unified-exec
---

# Codex 卷五执行说明

> 本文件不是灵感备忘，而是 **Codex 新卷五《统一执行子系统怎样把动作变成可管理会话》** 的执行协议。
> 后续如果继续修卷五，默认先以本文件作为卷级边界与顺序基线。

---

## 一、卷五唯一主问题

卷五唯一主问题是：

> **当 Codex 真正进入动作执行时，为什么它不是一个 shell wrapper，而是一套 sessionful、approval-aware、streamable、可对账的统一执行子系统？**

因此卷五期间：

- 不得把卷五写成 shell / CLI 命令教程
- 不得把卷五写成环境配置、sandbox 后端罗列或进程 API 手册
- 不得回退去重讲卷四控制面语义
- 不得提前吃掉下一卷 capability system / MCP / skills / plugins 平台层

卷五最后必须收束为：

> **Codex 的动作执行，不是一次命令调用，而是一套被统一装配、可被批准、可被流式观察、可统一收尾、可持续对账的执行会话系统。**

---

## 二、卷五当前唯一有效结构

卷五当前有效结构固定为 5 篇：

1. 为什么 `exec.rs` 和 unified-exec 不是一回事
2. `UnifiedExecHandler` 和 `UnifiedExecRuntime` 是怎么把动作装成执行会话的
3. 为什么 approval、sandbox、policy 不是执行外围，而是在执行前就进入主链
4. 输出为什么先进入 transcript，而不是直接变成“最终结果”
5. 为什么 process store 不是最终权威源，而 unified-exec 更像 execution control plane

对象主线顺序固定为：

> **primitive exec 边界 → 会话装配入口 → 执行前控制链 → transcript 输出语义 → process store / end event / execution control plane 收口**

---

## 三、文章角色分工

### 01｜总边界篇
- 负责切开 `exec.rs` 与 unified-exec subsystem
- 负责把卷五从“执行命令”提升为“执行会话系统”
- 不承担 approval / transcript / process store 的细节展开

### 02｜会话装配篇
- 负责解释 handler / runtime 怎样把动作请求装成正式执行会话
- 它是卷五的入口机制篇
- 不承担输出流与终态语义的完整解释

### 03｜执行前控制链篇
- 负责把 approval / sandbox / policy / runtime route 写成执行前主链
- 不把 guardian / review 专题扩进来
- 不写成权限系统总览

### 04｜输出语义篇
- 负责把 transcript / delta / aggregated output 写成 unified-exec 的主语义轨
- 解释流式输出为什么不是“最终字符串先验存在”
- 不承担 process store 权威边界的最终收口

### 05｜卷尾收口篇
- 负责把 transcript / process store / end event 的关系压回 execution control plane
- 负责把卷五最后收成“可管理执行会话”这个 retained takeaway
- 它必须最后写

---

## 四、执行顺序

卷五默认采用：

> **前 3 篇先立边界与入口，中后段再讲输出与对账，卷尾单独收口。**

固定顺序：

1. **01 → 02 → 03**
2. **04** 在 03 稳定后启动
3. **05 最后单独启动**

不建议：

- 从 transcript 或 process store 直接起跑
- 把卷五写成细碎函数导读
- 让 approval / sandbox 过早吞掉全卷主问题
- 在 05 里重新展开前面已建立的执行入口细节

---

## 五、当前真实状态（2026-04-13 核对）

已核对到：

- `README-writing-cards` 已存在
- 本文件补齐后，卷级执行说明也已存在
- 正文文件当前尚未起稿

因此卷五当前应被理解为：

> **卷级规划已建立，接下来应按 orch 路线先起第一批正文，而不是继续扩讨论。**

---

## 六、后续修订优先级

如果下一轮继续修卷五，默认顺序：

1. 先看 01 是否把 primitive exec 和 unified-exec subsystem 切干净
2. 再看 02 / 03 是否把“会话装配”与“执行前控制链”分开
3. 再看 04 是否把 transcript 立成主语义轨
4. 最后看 05 是否真正把整卷压成 execution control plane 结论
