---
title: Codex guidebook 05｜unified-exec 执行子系统
date: 2026-04-12
status: draft
---

# 05｜unified-exec 执行子系统

## 先给结论

如果只从名字看，很多人会把 unified-exec 理解成“比 `exec.rs` 更高级一点的执行封装”。但从源码角色看，这个判断太弱了。

更准确的结论是：

> **unified-exec 不是普通 shell 执行封装，而是一个 agent-facing、sessionful、approval-aware、streamable、recoverable、protocolized 的执行子系统。**

它解决的不只是“把命令跑起来”，而是整条链：

- 请求如何进入执行面
- approval 和 sandbox 如何介入
- 进程如何会话化
- 输出如何进入 transcript
- 成功/失败如何发出统一 end event
- 进程存储如何和真实生命周期对账

这章就沿这条主线展开。

---

## 1. 为什么 `exec.rs` 和 unified-exec 不是一回事

先把一个最容易混淆的边界讲清楚。

### `exec.rs`
它更接近执行 primitive 层。

它负责的是：

- 底层执行
- sandbox / policy 相关分支
- 把命令真正交给系统跑起来

它很重要，但它更像一条执行原语链。

### unified-exec
它在这条原语链之上，再加了一整层产品语义和会话语义：

- process id
- 长生命周期会话
- streaming output
- transcript delta
- approval / guardian / cache
- success / failure end event
- process store reconciliation

所以更准确地说：

- `exec.rs` 解决“怎么执行”
- unified-exec 解决“怎么把执行变成一个可被 agent 使用和管理的 runtime 子系统”

这就是两者真正的区别。

---

## 2. 入口层：`UnifiedExecHandler::handle`

unified-exec 真正进入 runtime 的第一站，不是直接 spawn，而是 handler。

这个 handler 负责的事情包括：

- 解析模型传来的执行参数
- 处理 shell / cwd / command 相关装配
- 分配 process id
- 准备 approval / runtime 所需的上下文

所以它不是“调用一下 runtime.run”的薄转发，而更像：

> **执行请求的入口装配线。**

到这里，统一执行子系统已经和普通命令执行分开了：一个模型 tool call，会先被包装成一个具有明确 process/session 身份的执行请求，而不是直接走到底层 shell。

---

## 3. `UnifiedExecRuntime` 才是 approval / sandbox / run policy 的交汇点

如果 handler 负责装配，那么 runtime 负责的就是更难的事情：

- 这次执行要不要 approval
- 已经批准过的键能不能复用
- guardian 能不能代审
- 网络访问是否需要额外确认
- sandbox 失败后如何处理
- 最终应该走本地执行还是环境后端执行

这也是 unified-exec 真正区别于普通 exec 的地方：

> **它不是“先执行，再处理周边语义”，而是在执行前就把 approval/policy/runtime 路由纳入主链。**

所以 `UnifiedExecRuntime` 更像统一执行控制器，而不是一个简单 executor。

---

## 4. spawn 边界：`open_session_with_exec_env`

执行子系统真正把请求落到进程层时，关键边界函数之一是 `open_session_with_exec_env(...)`。

它的意义不只是“打开一个 session”，而是：

- 决定这次执行到底在哪个环境里跑
- 处理本地 PTY 与远端 exec backend 的分支
- 把 exec environment 统一成 unified-exec 可持续管理的会话

也就是说，到了这里，系统已经不只是拿一段命令去执行，而是在做：

> **把执行请求适配成一个可持续跟踪的进程会话。**

从架构上看，这一步是 unified-exec 从“请求装配”进入“真实进程生命周期”的桥。

---

## 5. 为什么 unified-exec 是 sessionful subsystem

要证明 unified-exec 是一个真正的子系统，而不是薄封装，看 process session/store 模型最明显。

系统会做这些事：

- 分配 process id
- 把 process 存入 store
- 跟踪它的活跃状态
- 在合适时机清理/修正存储

这说明 unified-exec 的对象不是“一次命令调用”，而是：

> **一个有身份、有生命周期、可被追踪的执行会话。**

也正因为如此，后面才能成立：

- 多次查看状态
- 输出持续流式回传
- 最终 end event 统一封装
- 进程真实状态与 store 状态持续对账

这已经明显超出了简单 shell wrapper 的范畴。

---

## 6. 输出为什么先进入 transcript，而不是直接作为“最终结果”

unified-exec 最值得讲的一条链，是输出生命周期。

它不是简单把 stdout/stderr 拼起来直接返回，而是会经过这些步骤：

1. 从进程接收原始字节流
2. 做 UTF-8 安全切流
3. 形成 transcript delta
4. 写入 transcript / output tracking
5. 最终再决定聚合输出是什么

这说明系统对“输出”的理解，不是最终字符串，而是：

> **一个正在进行中的事件流。**

也正因为统一执行会走 transcript，所以它才能和整体 agent runtime 对上：

- 输出不仅是给用户看
- 也是系统内部可继续理解和处理的事件材料

因此，transcript 在 unified-exec 里是主语义轨，而不是附属日志。

---

## 7. `process_chunk` 与 UTF-8 边界：为什么这里要这么小心

统一执行的输出链上，有几个很小但非常关键的边界函数。

比如 `process_chunk(...)` 和 UTF-8 前缀切分相关逻辑，它们的任务不是做花哨格式化，而是保证：

- 输出被正确切片
- 不会把 UTF-8 字符拆坏
- transcript delta 和最终视图能稳定对齐

这类函数很典型地体现了 Codex 的整体风格：

- 主骨架不一定复杂得惊人
- 但真正的正确性，往往落在这些小边界函数上

unified-exec 能把 streaming output 做成可靠子系统，靠的不是一个“大输出管理器” alone，而是这批小边界函数共同维持。

---

## 8. 终态为什么要统一封装成 end event

如果统一执行只是跑命令，那退出码出来就差不多了。但 unified-exec 不是这么做的。

它会把执行结束包装成统一 end event，而且 success 和 failure 都有对应的终态封装路径。

这样做的意义在于：

- 输出流结束只是一个低层事实
- 但系统需要的是一个稳定的、可协议化的“执行结束语义”

统一 end event 之后，上层才能稳定处理：

- 这次 exec 是成功还是失败
- transcript 上该怎样收尾
- UI/agent/runtime 应该看到什么

所以 end packaging 不是装饰层，而是 unified-exec 作为子系统闭环的一部分。

---

## 9. transcript 为什么比 process store 更接近最终语义真相

这是 unified-exec 里一个很重要的判断。

系统并不是把 process store 当成“最终输出真相源”，而更倾向于把 transcript 看成最终聚合输出的主要依据。

原因很自然：

- process store 记录的是进程态和生命周期态
- transcript 记录的是对外真实可见的执行语义轨迹

因此，在最终聚合输出时，系统更信 transcript，而不是某个单独的进程缓存对象。

这和前面 rollout 在正文恢复里的角色有点类似：

- 不是所有低层存储都等于最终语义真相
- 真正的真相，往往在更高一层的语义投影上

---

## 10. `refresh_process_state`：为什么 process store 不是权威源

`refresh_process_state(...)` 这类函数的存在，本身就在说明一个事实：

> **process store 不是无条件可信的真相源，它需要不断和真实进程生命周期对账。**

如果 store 本身就是唯一 truth，就不需要专门 reconciliation。

系统之所以要 refresh / reconcile，是因为 unified-exec 接受这样的现实：

- 真实进程状态在外面变化
- store 是系统内部的投影和管理面
- 两者之间可能出现延迟或偏差

因此，统一执行的正确性也不是靠“永远只信 store”来保证，而是靠：

- 真实进程
- transcript
- process store
- end event

这几个面一起形成闭环。

---

## 11. unified-exec 和环境 / policy / sandbox 的关系

统一执行不是孤立存在的。它还嵌在更大的执行边界里：

- exec environment
- exec policy
- sandbox
- environment manager / remote exec backend

所以可以把它理解成中间层：

- 再往下，是执行原语和环境约束
- 再往上，是 agent tool call 和用户可见的运行态

它把这两边真正接了起来。

因此 unified-exec 最合理的定位不是“执行工具”，而是：

> **agent-facing execution control plane**

它既懂运行时环境约束，又懂上层 session/protocol 语义。

---

## 12. 这一章最重要的判断

读完这章，应该稳定这些结论。

### 判断 1：`exec.rs` 和 unified-exec 不是一回事
前者是原语层，后者是会话化执行子系统。

### 判断 2：`UnifiedExecHandler` 是入口装配线
它把模型请求装成真正的执行会话请求。

### 判断 3：`UnifiedExecRuntime` 是 approval/policy/run 的交汇点
它决定执行到底怎么被允许、怎么被路由、怎么被运行。

### 判断 4：spawn 边界函数把请求变成可持续管理的进程会话
不是简单地起个命令。

### 判断 5：unified-exec 是 sessionful subsystem
process id、store、watch、cleanup 共同证明它不是一次性调用。

### 判断 6：输出主线先进入 transcript
而不是直接把 stdout/stderr 当最终结果。

### 判断 7：end event 是统一终态语义
不是退出码的包装纸。

### 判断 8：process store 不是绝对真相源
它需要和真实进程生命周期持续对账。

### 判断 9：unified-exec 是 agent-facing execution control plane
这才是它在系统里的真正层级。

---

## 13. 下一章该接什么

到这里，系统主干已经有了：

- 分层
- 状态恢复
- app-server thread/turn 主线
- turn-history 语义层
- unified-exec 执行子系统

最后还缺的一块，是那些不直接属于主 runtime spine，但又非常影响系统行为的扩展层和高级子系统：

- plugins / skills / MCP / apps / connectors
- review / guardian / realtime / collab / memories / agents

所以，下一章不再追某一条单独主链，而是收尾整套系统的扩展面：

> **capability system 与高级子系统。**

---

## 相关阅读

- 想先回看 unified-exec 挂在哪条系统总图里：[[01-系统总图与分层]]
- 想继续看 capability / guardian 如何与执行审批边界交叉：[[06-capability与高级子系统]]、[[09-review工作流与guardian审查基础设施]]
- 想直接看执行链函数与顺序：[[11-关键函数索引]]、[[12-调用链索引]]

## 关键文件

- `codex-rs/core/src/tools/handlers/unified_exec.rs`
- `codex-rs/core/src/tools/runtimes/unified_exec.rs`
- `codex-rs/core/src/unified_exec/mod.rs`
- `codex-rs/core/src/unified_exec/process_manager.rs`
- `codex-rs/core/src/unified_exec/process.rs`
- `codex-rs/core/src/unified_exec/async_watcher.rs`
- `codex-rs/core/src/exec.rs`
- `codex-rs/core/src/exec_env.rs`
- `codex-rs/core/src/exec_policy.rs`
