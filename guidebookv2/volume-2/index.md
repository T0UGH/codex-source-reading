---
title: 卷二｜runtime core 主线
status: draft
updated: 2026-04-14
---

# 卷二｜runtime core 主线

卷二回答：一条请求怎么真正进入 Codex 的 runtime core，并最终闭合成一轮可以持续推进的工作回合。

## 目录

1. [Codex 新卷二 01：一次请求怎么进入 Codex 的 runtime 主线](./2026-04-12-Codex-新卷二-01-一次请求怎么进入-Codex-的-runtime-主线-v1.md)
2. [Codex 新卷二 02：`ThreadManager`、`CodexThread`、`Codex` 怎么接成一条 runtime 主工作链](./2026-04-12-Codex-新卷二-02-ThreadManager-CodexThread-Codex-怎么接成主工作链-v1.md)
3. [Codex 新卷二 03：当前工作面是怎么被组织出来的](./2026-04-12-Codex-新卷二-03-当前工作面是怎么被组织出来的-v1.md)
4. [Codex 新卷二 04：thread 和 turn 为什么不是两个平行名词](./2026-04-12-Codex-新卷二-04-thread-和-turn-为什么不是两个平行名词-v1.md)
5. [Codex 新卷二 05：系统怎么判断这一轮要不要调用能力](./2026-04-12-Codex-新卷二-05-系统怎么判断这一轮要不要调用能力-v1.md)
6. [Codex 新卷二 06：动作结果怎么重新回到当前工作回合](./2026-04-12-Codex-新卷二-06-动作结果怎么重新回到当前工作回合-v1.md)
7. [Codex 新卷二 07：一轮工作回合什么时候继续，什么时候收口](./2026-04-12-Codex-新卷二-07-一轮工作回合什么时候继续什么时候收口-v1.md)
8. [Codex 新卷二 08：把整条 runtime core 主线重新压成稳定运行图](./2026-04-12-Codex-新卷二-08-把整条-runtime-core-主线重新压成稳定运行图-v1.md)

## 辅助文件

本卷对应的写作卡片与执行说明也已一并迁入仓库，但不放进主导航。
