---
title: 卷五｜统一执行子系统
status: draft
updated: 2026-04-14
---

# 卷五｜统一执行子系统

卷五回答：Codex 怎么把动作组织成正式 execution session，而不是只做一次性命令调用。

## 目录

1. [为什么 `exec.rs` 和 unified-exec 不是一回事](./2026-04-13-Codex-卷五-01-为什么-exec-rs-和-unified-exec-不是一回事-v1.md)
2. [`UnifiedExecHandler` 和 `UnifiedExecRuntime` 是怎么把动作装成执行会话的](./2026-04-13-Codex-卷五-02-UnifiedExecHandler-和-UnifiedExecRuntime-是怎么把动作装成执行会话的-v1.md)
3. [为什么 approval、sandbox、policy 不是执行外围，而是在执行前就进入主链](./2026-04-13-Codex-卷五-03-为什么-approval-sandbox-policy-不是执行外围而是在执行前就进入主链-v1.md)
4. [输出为什么先进入 transcript 而不是直接变成最终结果](./2026-04-13-Codex-卷五-04-输出为什么先进入-transcript-而不是直接变成最终结果-v1.md)
5. [为什么 process store 不是最终权威源，而 unified-exec 更像 execution control plane](./2026-04-13-Codex-卷五-05-为什么-process-store-不是最终权威源而-unified-exec-更像-execution-control-plane-v1.md)