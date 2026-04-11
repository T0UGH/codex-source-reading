# Codex 源码拆解 08：sandbox、execpolicy、approval 的分工

## 这篇看什么

回答：**Codex 为什么不是裸奔 agent？sandbox、execpolicy、approval 到底各管什么？**

## 先给结论

Codex 在执行边界上至少有三层：

1. **sandbox**：实际执行环境边界
2. **execpolicy**：策略判断与规则评估
3. **approval**：用户/系统层面的动作授权门禁

三者不是重复，而是分工：

- sandbox 负责“能不能在这个环境里跑”
- execpolicy 负责“从规则上是否允许这样跑”
- approval 负责“这次动作要不要经过确认”

## 关键代码路径

- `codex-rs/core/src/tools/sandboxing.rs`
- `codex-rs/sandboxing/`
- `codex-rs/linux-sandbox/`
- `codex-rs/windows-sandbox-rs/`
- `codex-rs/execpolicy/Cargo.toml`
- `codex-rs/execpolicy/src/policy.rs`
- `codex-rs/execpolicy/src/execpolicycheck.rs`

## 关键判断

### 1. `core/src/tools/sandboxing.rs` 是统一编排层
这里不是平台实现，而是：
- cached approvals
- approval requirement
- sandbox override / bypass
- `Approvable` / `Sandboxable` / `ToolRuntime` 等 trait

这说明它想统一表达：
> 一个工具动作在被执行前，要先经过什么样的策略与批准判断。

### 2. sandbox 是环境边界，不只是布尔开关
平台实现拆到了：
- `sandboxing`
- `linux-sandbox`
- `windows-sandbox-rs`

这说明真正的执行环境约束是平台相关的，不是简单 if/else。

### 3. execpolicy 是单独的策略规则系统
它不是 sandbox 的别名。

从 `execpolicy` crate 能看出它在做：
- command prefix rules
- network allow/forbid
- overlay merging
- host executable resolution
- heuristic fallback

这更像：
> 执行策略解释器 / evaluator

### 4. approval 是运行时门禁
根据 approval mode、sandbox mode、policy 限制等，系统会判断：
- skip
- needs approval
- forbidden

说明 approval 不是环境能力，而是 **decision gate**。

## 设计判断

### 1. 这是清晰的三层边界
- sandbox：环境限制
- execpolicy：规则限制
- approval：交互式授权限制

### 2. 这套分层很工程化
因为 agent 系统里最容易混掉的就是：
- 物理能不能跑
- 规则允不允许跑
- 当前要不要先问用户

Codex 至少没有把这三件事糊成一个开关。

### 3. 这也解释了为什么 Codex 的安全边界不是只靠“请谨慎”
它在结构上确实有独立层。

## 当前边界判断

- `sandboxing`：执行环境边界
- `execpolicy`：策略规则评估
- `approval`：运行时动作门禁
- `core/src/tools/sandboxing.rs`：三者汇流的统一编排层

## 还没确认的点

1. macOS 平台的 sandbox 主链还没单独展开。
2. execpolicy 与 MCP/tool approvals 的更细关系值得后续再拆。
3. `UnlessTrusted`、`Granular` 等模式的真实整体行为还可单列一篇。

## 下一步建议

继续写：
- `state / rollout / session 恢复链`
- `app-server / SDK / embedding 架构`
