# Codex 源码拆解 14：skills 与 MCP 的依赖路径

## 这篇看什么

回答：**skills 和 MCP 是不是一回事？它们之间到底怎么互相影响？**

## 先给结论

不是一回事，而且边界比表面上更清楚：

1. `skills` 是 **提示/流程资产层**
2. `MCP` 是 **外部工具/连接器接入层**
3. 两者的主交点不是“skills 调 MCP runtime”，而是：
   - **plugins 改变 effective skill roots**
   - **plugins 也参与生成 `to_mcp_config(...)`**
4. 换句话说，真正的桥不是 `skills ↔ MCP`，而是 **plugins 同时影响了两边**

一句话：

> skills 和 MCP 不是父子关系，而是共享一部分 plugin/config 基础设施的两条并行能力线。

## 关键代码路径

- `codex-rs/core/src/skills.rs`
- `codex-rs/core/src/mcp.rs`
- `codex-rs/core/src/codex.rs`
- `codex-rs/core/src/thread_manager.rs`
- `codex-rs/instructions/src/user_instructions.rs`

## 证据 1：skills 加载路径独立存在

`core/src/codex.rs` 启动 session 时：

- `plugins_manager.plugins_for_config(&config)`
- `plugin_outcome.effective_skill_roots()`
- `skills_load_input_from_config(...)`
- `skills_manager.skills_for_config(...)`

每个 turn 也会再按当前 config 重新算：

- `plugins_for_config(&per_turn_config)`
- `effective_skill_roots()`
- `skills_for_config(...)`

这说明 skills 是一条独立的注入链：

`plugins outcome` → `skill roots` → `skills manager` → `turn_skills outcome`

## 证据 2：skills 最终进的是 instructions / developer sections

`core/src/codex.rs` 里：

- `turn_skills.outcome.allowed_skills_for_implicit_invocation()`
- `render_skills_section(...)`
- push 到 `developer_sections`

`instructions/src/user_instructions.rs` 里还定义了 `SkillInstructions`，格式是：

```text
<skill>
<name>...</name>
<path>...</path>
...
</skill>
```

这说明 skills 的主要产品语义是：

- 注入上下文
- 约束行为
- 引导流程
- 支持 implicit invocation / telemetry / dependency asking

它首先是 prompt/runtime guidance，不是工具 transport。

## 证据 3：MCP manager 是另一条链

`core/src/mcp.rs` 很干净：

- `configured_servers(config)`
- `effective_servers(config, auth)`
- `tool_plugin_provenance(config)`

关键点在于它都依赖：

- `config.to_mcp_config(self.plugins_manager.as_ref())`

说明 MCP 那边的主链是：

`config + plugins` → `mcp_config` → effective/configured servers

它处理的是服务器配置、鉴权态、生效 server，以及 connector/tool provenance。

## 证据 4：交点在 plugins，而不在 skills runtime

现在能看到的最核心交点有两个：

### 交点 A：plugins 决定 effective skill roots
`plugin_outcome.effective_skill_roots()`

### 交点 B：plugins 参与 `to_mcp_config(...)`
`config.to_mcp_config(plugins_manager.as_ref())`

也就是说：

- plugin 可以带来 skills
- plugin 也可以带来 MCP server / connector 相关配置

所以真实关系是：

```text
plugins
├─ 影响 skills roots / capability summary
└─ 影响 mcp config / tool provenance
```

而不是：

```text
skills -> mcp
```

## 证据 5：exec_command 的 implicit skill invocation 只是 telemetry / guidance，不是直接调 MCP

`skills.rs` 里的 `maybe_emit_implicit_skill_invocation(...)`：

- 基于 command + workdir 检测隐式技能命中
- 记录 telemetry
- 避免重复注入

这里没有直接调 MCP manager。

说明 skill 命中的主要作用仍是：
- 行为提示
- 统计
- turn 内技能上下文

而不是直接变成 MCP 调用。

## 设计判断

### 1. skills 是“怎么做”的资产

它表达：
- 流程
- 规则
- 依赖
- 经验
- 指令片段

### 2. MCP 是“能调用什么外部能力”的接入面

它表达：
- server
- connector
- auth
- tool provenance
- runtime connection manager

### 3. plugin 才是更上游的装配点

当前源码里，plugin 才是最值得关注的 capability packaging 单位。

因为它同时能影响：
- skills
- MCP
- capability summary
- app/connectors 展示

### 4. 所谓“skills ↔ MCP dependency path”更准确应改写为“plugin-mediated coupling”

如果后面写 guide，我会建议直接这么叫。

不然很容易误导成：
- skills 依赖 MCP runtime
- 或 MCP 直接载入 skills

这两个判断目前都不准确。

## 当前边界判断

- `skills`：提示/流程/经验资产层
- `mcp`：外部工具与连接器接入层
- `plugins`：共同上游装配点
- `instructions`：skills 的主要投影面之一
- `turn_skills`：每个 turn 的技能上下文快照

## 后续值得继续挖的点

1. plugin manifest 到底如何同时声明 skills 和 mcp 能力
2. capability summary 如何把 plugin / skill / connector 汇总到 UI 层
3. skills dependency asking（环境变量）和外部工具可用性之间是否会进一步联动
