---
title: 为什么 process store 不是最终权威源，而 unified-exec 更像 execution control plane
date: 2026-04-13
status: draft
tags:
  - Codex
  - guidebook
  - volume-5
  - unified-exec
  - process-store
  - execution-control-plane
---

# 为什么 process store 不是最终权威源，而 unified-exec 更像 execution control plane

## 读者问题

如果 unified-exec 已经维护了一套 process store，为什么我们还不能把它理解成“执行结果的最终真相库”？

再往前走一步，为什么到了卷五收口时，更准确的判断反而是：**unified-exec 更像一层 execution control plane，而不只是一个管理进程条目的地方？**

## 结论

结论先写在前面：

> **process store 记录的是执行会话的可寻址状态，并持续和真实进程生命周期对账，但它不是最终语义的唯一权威源。真正成立的是，unified-exec 把一次动作组织成了一套可批准、可流式观察、可统一收尾、可持续对账的执行会话系统。**

所以卷五最后要留下的，不是“Codex 很会管进程”，而是：

> **Codex 在管理执行会话。**

process store 只是这套会话系统中的托管层；transcript 提供正在发生的语义轨迹；end event 把完成态封装成稳定协议。三者合在一起，才构成 unified-exec 的 execution control plane。到这里，卷五也该把 retained takeaway 压成一句：**Codex 管理的是执行会话，而不是命令调用。**

---

## 先用白话解释几个术语

### process store 是什么

这里的 **process store**，可以先把它理解成：

> **系统手里那本“活会话登记簿”。**

哪些进程还活着、能不能继续写 stdin、后面还能不能 poll，它负责让这些执行会话可以被再次找到、再次操作。

### transcript 是什么

这里的 **transcript**，可以先把它理解成：

> **执行过程的事件账本。**

输出不是等命令结束后一次性生成，而是在运行中持续变成 delta、持续被观察、持续被记录。

### end event 是什么

这里的 **end event**，可以先把它理解成：

> **统一收尾回执。**

不管成功还是失败，系统最后都要把本次执行会话收口成一个上层能稳定消费的结束事件，而不是把零散内部状态直接裸露出去。

把三者放在一起看，意思就清楚了：

- store 负责托管
- transcript 负责呈现过程
- end event 负责协议化收尾

它们不是互相替代，而是共同服务于一条执行会话生命周期。

---

## 第一层：为什么 process store 不是最终权威源

最直接的证据，不是一句抽象定义，而是系统里专门存在 `refresh_process_state(...)` 这种对账动作。

既然还需要对账，就说明 store 里的记录并不自动等于最终真相；它必须不断回看真实进程是否已经退出，再据此修正自己的条目。

这件事的含义很重要：

> **process store 更像 live-session registry，而不是不可置疑的 source of truth。**

它保存的是“系统当前如何托管这些执行会话”，而不是“这些会话全部语义已经天然完备”。

这也是为什么 `from_spawned(...)` 与 `store_process(...)` 连起来看，会呈现出一条很清楚的接力链：

- 前者先把 raw spawn 包成 unified-exec session
- 后者再把这个 session 纳入 manager 的长期托管世界

也就是说，store 的强项是把会话纳入管理面，而不是独自定义会话真相。

---

## 第二层：为什么 transcript 比单独的 process store 更接近执行语义

上一篇已经立住：unified-exec 的输出首先进入 transcript，而不是先验地存在于某个“最终字符串”里。

这意味着，对外真正可观察的执行语义，并不是一条进程表记录，而是一条持续产生的事件流。

因此到了收尾阶段，系统会优先从 transcript 解析 aggregated output；只有 transcript 不足时，才回退到 fallback。这个优先级本身就在说明：

> **对“这次执行到底向外呈现了什么”，transcript 比 process store 更接近语义真相。**

换句话说：

- store 解决的是“还能不能找到这次会话”
- transcript 解决的是“这次会话实际说了什么、暴露了什么、被观察到了什么”

前者偏托管，后者偏语义。

---

## 第三层：为什么 end event 才是统一终态的外部出口

如果系统只是在管命令，那么拿到退出码，事情差不多就结束了。

但 unified-exec 不是这样收尾的。

无论成功还是失败，它都要经过统一的 end-event 封装路径，把 transcript 优先的最终输出、执行上下文、完成状态一起打包，再发射成稳定的 `ExecCommandEnd`。

这一步的价值在于：

> **内部生命周期事实，必须先被翻译成外部协议事实，执行会话才算真正结束。**

所以 end event 的角色，不是“顺手发一个通知”，而是把分散在：

- 进程生命周期
- 输出流
- 失败说明
- 上下文信息

这些内部材料，压成一份上层可以持续依赖的终态协议。

也正因为如此，成功与失败都不是旁路异常，而是同构终态。

---

## 第四层：把 transcript、process store、end event 压回一条控制链

到这里，三者的关系就应该这样理解：

- **transcript** 负责执行中的语义流，让会话可以被流式观察
- **process store** 负责会话托管与后续寻址，让活会话可以被继续操作，并和真实生命周期持续对账
- **end event** 负责统一终态封装，让会话最终以稳定协议收尾

这三层共同说明，unified-exec 真正在做的，不是“帮你调一次命令”，而是：

> **把一次动作从请求入口、执行过程到结束回执，整个组织成受控会话。**

所以把 unified-exec 理解成 process registry，会把它看窄；把它理解成 shell wrapper，会把它看浅。

更准确的说法是：

> **它是一层 execution control plane。**

这里说的 control plane，不是平台级插件总线，也不是更外层的 MCP 世界；在卷五范围内，它只指一件事：

> **Codex 已经把动作执行提升成一套可控制、可观察、可收尾、可对账的会话管理面。**

---

## 第五层：卷五最终要留下什么记忆点

在真正收口前，可以先把卷五前四篇重新挂回这一篇。卷五其实是按下面这五步一路压过来的：

1. **先切开 `exec.rs` 和 unified-exec**：证明这不是“底层执行换个名字”。
2. **再看 handler / runtime 怎样建立执行会话**：证明系统先管理的是 session，而不是命令字符串。
3. **再看 approval / sandbox / policy 怎样进入主链**：证明执行前控制链从一开始就在系统里，而不是外围补丁。
4. **再看 transcript 为什么是主语义轨**：证明输出首先属于执行语义，不只是最后拼出来的文本。
5. **最后才回到 process store / end event / execution control plane**：把前面这些东西压成一条统一的执行控制链。

卷五走到这里，读者最稳的 takeaway 应该是：

> **Codex 的动作执行不是一次命令调用，而是一套可批准、可流式观察、可统一收尾、可持续对账的执行会话系统。**

也因此，process store 虽然重要，却不应被误认成最终权威源。

它是 execution control plane 里的一个关键托管层；但真正把 Codex 拉出“命令执行器”心智的，是 unified-exec 对整条执行会话生命周期的统一管理。

---

## 收口

如果只盯着 store，会把 unified-exec 读成进程管理模块；如果只盯着 spawn，会把它读成命令封装器；如果只盯着 output，又会把它读成流式日志系统。

真正完整的看法是：

> **transcript 提供过程语义，process store 提供会话托管，end event 提供统一终态，而 unified-exec 把这三者压进同一条执行控制链。**

这也是卷五最后的收束：

> **Codex 管理的不是一次命令调用，而是一整个执行会话。**

到这里，卷五就该收住。下一卷再自然转入：能力系统怎样建立在执行面之上，把 Codex 长成平台。
---

## 卷内导航

- 上一篇：[《输出为什么先进入 transcript，而不是直接变成“最终结果”》](./2026-04-13-Codex-卷五-04-输出为什么先进入-transcript-而不是直接变成最终结果.md)
- 回到本卷入口：[本卷导读](./index.md)
- 这是本卷卷尾，读完建议先回到本卷入口再决定是否跳卷。

