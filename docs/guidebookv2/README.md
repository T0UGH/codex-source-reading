---
title: Codex 源码导读手册 v2
date: 2026-04-14
status: draft
---

# Codex 源码导读手册 v2

> `guidebookv2/` 是当前这套手册的**正式阅读入口**，也是整本书按 **6 个认知台阶** 重组后的主线版本。

这里不再按旧目录树或源码目录平铺，而是优先按读者理解 Codex 所需要跨过的认知坡度来组织全书。

## 先读这一句

> **卷一到卷六不是材料分桶，而是一条连续问题链：先看清这是什么系统，再看它怎么续跑、怎么投影控制面、怎么把执行动作装成执行会话，最后再进入高级子系统与 future-looking 缺口。**

## 当前状态

`guidebookv2/` 现在已经形成完整六卷主线：

1. [卷一｜Codex 系统全景导论](./volume-1/index.md)
2. [卷二｜状态、持久化与恢复](./volume-2/index.md)
3. [卷三｜app-server、thread 与 turn 主线](./volume-3/index.md)
4. [卷四｜统一执行子系统](./volume-4/index.md)
5. [卷五｜能力系统与高级子系统](./volume-5/index.md)
6. [卷六｜附录、微缺口与继续深挖入口](./volume-6/index.md)

说明：
- `guidebookv2/` 是当前默认导航与默认阅读入口
- 仓库原有 guidebook / source notes / boundary notes 仍保留在 repo 中，主要作为迁移素材与证据层
- 后续站点入口、README、导航与公开阅读链，默认都以 `guidebookv2/` 为主线

---

## 六个认知台阶分别回答什么

### 卷一：Codex 是什么系统
回答：**Codex 到底是什么系统，它的核心对象和基本分层是什么。**

### 卷二：状态与恢复为什么能让系统持续工作
回答：**Codex 为什么不是跑完就散，而是能持续、恢复、继续推进的系统。**

### 卷三：控制面主线是怎么成立的
回答：**app-server、thread、listener、turn、turn-history 怎样被组织成一个可连接、可恢复、可 replay 的控制面。**

### 卷四：执行子系统是怎么成立的
回答：**执行动作怎样被统一装成 execution session，而不是单点命令调用。**

### 卷五：高级子系统怎样把 Codex 推向平台层
回答：**模型传输、后端边界、review、guardian、realtime、collab、memories 等系统怎样把 Codex 推向更高层 runtime 组织。**

### 卷六：从哪里继续深挖
回答：**哪些索引、开放问题和微缺口最值得作为下一轮继续阅读入口。**

---

## 推荐阅读方式

### 如果你第一次读这套手册
建议这样进入：

- [卷一｜Codex 系统全景导论](./volume-1/index.md)
- [卷二｜状态、持久化与恢复](./volume-2/index.md)
- 然后按卷三到卷六顺序继续

### 如果你已经熟悉 coding agent，想直接看 runtime 主线
可以先读：

- [卷二｜状态、持久化与恢复](./volume-2/index.md)
- [卷三｜app-server、thread 与 turn 主线](./volume-3/index.md)
- [卷四｜统一执行子系统](./volume-4/index.md)

### 如果你更关心平台、高级能力与未来缺口
可以直接进入后半段：

- [卷五｜能力系统与高级子系统](./volume-5/index.md)
- [卷六｜附录、微缺口与继续深挖入口](./volume-6/index.md)

---

## 这版重组和旧结构的关键区别

这次重组最关键的变化，不是换了几个标题，而是把整套内容改成了明确的认知递进：

- 卷优先于文件堆叠
- 主问题优先于材料归档
- 认知递进优先于源码目录
- 公开阅读入口与仓库证据层分离
- 旧资产继续保留，但退出主导航

也就是说：

> **读者不是在逛 Codex 源码目录，而是在一层一层理解 Codex 这套系统。**

---

## 从这里继续

- 想先看全景：从 [卷一](./volume-1/index.md) 开始
- 想先看持续工作与恢复：从 [卷二](./volume-2/index.md) 开始
- 想先看高级子系统与未来缺口：从 [卷五](./volume-5/index.md) 开始
