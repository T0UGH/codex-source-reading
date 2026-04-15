---
title: 卷五｜统一执行子系统
status: draft
updated: 2026-04-14
---

# 卷五｜统一执行子系统

卷五回答：Codex 怎么把动作组织成正式 execution session，而不是只做一次性命令调用。

## 本卷读完后你应该得到什么

1. 知道 `exec.rs` 与 unified-exec 为什么是上下层，而不是平替
2. 知道 handler、runtime、session 启动之间的对象变化
3. 知道 approval、policy、sandbox、route 为什么都属于执行前主链
4. 知道 transcript、process store、end event 为什么共同构成 execution control plane

## 为什么这一卷必须现在读

如果你已经接受了 Codex 不是一个“会调命令的 agent 外壳”，这一卷就会继续把问题往前推一层：

> **Codex 也不是把命令丢给系统就完了；它会先把动作装成正式执行会话，再把批准、环境约束、输出语义与统一收尾压进同一条执行主链。**

## 本卷主问题链

- `exec.rs` 和 unified-exec 先切层级
- handler / runtime 再把动作装成执行会话
- approval / sandbox / policy 说明执行前控制链从一开始就在主链里
- transcript / process store / end event 继续说明 unified-exec 管的是执行会话，而不是单次命令

## 目录

1. [为什么 `exec.rs` 和 unified-exec 不是一回事](./2026-04-13-Codex-卷五-01-为什么-exec-rs-和-unified-exec-不是一回事.md)
2. [`UnifiedExecHandler` 和 `UnifiedExecRuntime` 是怎么把动作装成执行会话的](./2026-04-13-Codex-卷五-02-UnifiedExecHandler-和-UnifiedExecRuntime-是怎么把动作装成执行会话的.md)
3. [为什么 approval、sandbox、policy 不是执行外围，而是在执行前就进入主链](./2026-04-13-Codex-卷五-03-为什么-approval-sandbox-policy-不是执行外围而是在执行前就进入主链.md)
4. [输出为什么先进入 transcript 而不是直接变成最终结果](./2026-04-13-Codex-卷五-04-输出为什么先进入-transcript-而不是直接变成最终结果.md)
5. [为什么 process store 不是最终权威源，而 unified-exec 更像 execution control plane](./2026-04-13-Codex-卷五-05-为什么-process-store-不是最终权威源而-unified-exec-更像-execution-control-plane.md)

## 推荐阅读方式

- 如果你是第一次进卷五：按 01 → 02 → 03 → 04 → 05 顺着读，先立边界，再立会话，再立控制链
- 如果你最关心执行前控制：先读 02、03
- 如果你最关心输出与收尾：先读 04、05

## 本卷最后要留下的一句话

> **Codex 真正管理的不是一次命令调用，而是一整个可批准、可观察、可统一收尾、可持续对账的执行会话。**

## 卷间导流

- **卷四** 解释 runtime 怎样被暴露成控制面
- **卷五** 解释动作怎样被落成执行会话
- **卷六** 会继续回答：这些执行能力怎样向上长成审查、协作与更高层 runtime 组织