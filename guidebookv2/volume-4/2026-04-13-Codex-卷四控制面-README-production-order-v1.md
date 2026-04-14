---
title: Codex 卷四执行说明
date: 2026-04-13
status: draft
tags:
  - Codex
  - guidebook
  - production-order
  - volume-4
---

# Codex 卷四执行说明

> 本文件不是灵感备忘，而是 **Codex 新卷四《控制面怎么把 runtime 暴露出来》** 的执行协议。
> 后续如果继续修卷四，默认先以本文件作为卷级边界与顺序基线。

---

## 一、卷四唯一主问题

卷四唯一主问题是：

> **thread runtime 已经成立之后，Codex 怎样把它稳定投影成 TUI、SDK、远端调用者可观察、可控制、可恢复连接的控制面？**

因此卷四期间：

- 不得把卷四写成 app-server 文件目录导读
- 不得把卷四写成 request type 大表解释
- 不得把卷四重新退回“runtime 真正主体在哪里”这个已由卷二/卷四开篇回答过的问题
- 不得把卷四提前吃掉 unified-exec 的执行会话主线

卷四最后必须收束为：

> **Codex 已经长出正式控制面；app-server 不是另一套 runtime，而是把 core runtime 暴露成统一控制面 contract 的外露层。**

---

## 二、卷四当前唯一有效结构

卷四当前有效结构固定为 5 篇：

1. 为什么 app-server 不是另一套 runtime，而是建立在 core 之上的控制面 facade
2. listener task、协议投影与状态修正怎样把控制面撑起来
3. `ServerRequestResolved` 到底覆盖了什么控制面语义
4. 为什么 `DynamicToolCall` 不走 `ServerRequestResolved`
5. 为什么 TUI 越来越像跑在 app-server 之上，而不是直接抓 core

对象主线顺序固定为：

> **runtime owner / facade 边界 → 控制面成立机制 → request semantics 覆盖面 → 语义分叉点 → TUI over app-server 收口**

---

## 三、文章角色分工

### 01｜总边界篇
- 负责切开 `core / ThreadManager` 与 `app-server` 的关系
- 负责把 app-server 立成 control-plane facade
- 不承担 listener 细节或 request semantics 细分类

### 02｜控制面机制篇
- 负责解释 listener、协议投影、状态修正怎样共同把控制面撑起来
- 它是卷四真正的机制中轴
- 不承担 request shape 分类总表职责

### 03｜resolved 语义覆盖面篇
- 负责拆开 callback completion、thread-ordered resolved notification、对外协议语义
- 负责把 `ServerRequestResolved` 的覆盖边界收口
- 不完整展开 `DynamicToolCall`

### 04｜语义分叉篇
- 负责把 `DynamicToolCall` 从“未迁移洞”压成“有意的 item-lifecycle 分叉”
- 不回头重写 03 的 resolved 覆盖面总判断

### 05｜卷尾收口篇
- 负责把前四篇判断压回一个系统结论：为什么今天的 TUI 越来越像跑在 app-server 之上
- 负责把卷四收口成统一控制面 contract，而不是继续开新专题

---

## 四、执行顺序

卷四默认采用：

> **前二篇立边界与机制，中二篇收语义分类，最后单篇收口。**

固定顺序：

1. **01 → 02**
2. **03 → 04**
3. **05 最后单独启动**

不建议：

- 从 03 或 04 直接起跑
- 把 `DynamicToolCall` 提前写成卷四代表问题
- 在 02 里提前把 request semantics 吃掉
- 在 05 里重新展开前面已经解释过的细节链

---

## 五、当前真实状态（2026-04-13 核对）

已核对到：

- `卷四-01` 到 `卷四-05` 五篇正文文件都存在
- `README-writing-cards` 存在
- 本文件补齐后，卷级执行说明也已存在
- 当前五篇正文 frontmatter 仍均为 `status: draft`
- 尚未在已检查的 vault 文件中看到明确飞书交付痕迹

因此卷四当前应被理解为：

> **正文层已完整落盘，卷级执行层现已补齐，但整体仍处于 draft / 待验收状态。**

---

## 六、后续修订优先级

如果下一轮继续修卷四，默认顺序：

1. 先看 01 / 02 的边界是否足够稳
2. 再看 03 / 04 是否把“resolved 覆盖面”与“DynamicToolCall 分叉”切干净
3. 最后看 05 是否把卷四压成统一控制面结论

默认只做：

- 可读性增强
- 边界收紧
- 冗余压缩
- 章节导流与结尾收口强化

不默认做：

- 重新大改卷级结构
- 扩写到 unified-exec
- 回退去重讲 runtime core
