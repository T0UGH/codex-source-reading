# Codex 源码拆解 19：plugin 能力打包与 skills / MCP / apps 的关系

## 这篇看什么

回答：**plugin 到底是不是 Codex 里更上游的能力打包单元？它如何同时影响 skills、MCP 和 apps？**

## 先给结论

现在基本可以拍板：

1. plugin 是 **更上游的 capability packaging unit**
2. plugin manifest 的核心能力入口是：
   - `skills`
   - `mcpServers`
   - `apps`
3. `interface.capabilities` 更像 UI/展示元数据，不是主要 runtime 装配入口
4. 真正贯穿 runtime 的，是 `PluginCapabilitySummary`

一句话：

> plugin 不是“可选装饰层”，而是把 skills / MCP / apps 这三条能力线打包到一起的上游单元。

## 关键代码路径

- `codex-rs/core/src/plugins/manifest.rs`
- `codex-rs/core/src/plugins/manager.rs`
- `codex-rs/core/src/plugins/render.rs`
- `codex-rs/core/src/plugins/injection.rs`
- `codex-rs/app-server/src/codex_message_processor.rs`
- `codex-rs/codex-mcp/src/mcp/mod.rs`

## manifest 里真正重要的字段

从 `manifest.rs` 看，plugin manifest 核心字段是：

- `skills`
- `mcp_servers`
- `apps`
- `interface`

其中：

### `skills`
影响 effective skill roots。

### `mcpServers`
影响 MCP server config load / normalization。

### `apps`
影响 app connector / app summary。

### `interface.capabilities`
更多是描述这个 plugin “会什么”，偏展示层。

## 一个很重要的实现细节：三条线不是完全同一种 merge 方式
从 `plugins/manager.rs` 看：

### skills 是 additive 的
plugin skill roots =
- 默认 skill roots
- + manifest 显式声明 skill roots

### MCP / apps 更偏 override + fallback
- 如果 manifest 显式给 path，就优先用 manifest path
- 没有才退到默认 `.mcp.json` / `.app.json`

这个细节非常重要。

它说明 plugin packaging 虽然统一了三条能力线，但内部 merge 语义并不完全一致。

## `PluginCapabilitySummary` 才是 runtime 汇总层
运行时真正拿来汇总一个 plugin“能提供什么”的，是：

- `has_skills`
- `mcp_server_names`
- `app_connector_ids`

也就是说，Codex 实际上在 runtime 里把 plugin 看成：

> 一个可以同时拥有 skill、MCP server、app connector 的 bundle

这个结构比 `interface.capabilities` 更关键，因为它直接和：
- telemetry
- provenance
- injection
- UI summary

接上了。

## plugin 如何影响 MCP provenance
`codex-mcp` 那边会基于 plugin capability summaries 生成：

- `ToolPluginProvenance`

这会被用来：
- 给 connector / app 做 plugin attribution
- 帮助 UI/客户端理解“这个工具或 connector 来自哪个 plugin”

所以 plugin 不只是把 MCP server 配进来，还参与了后续 provenance 追踪。

## plugin 如何影响 prompt / model guidance
这个点很容易漏掉。

`plugins/render.rs` 和 `plugins/injection.rs` 说明：

- 插件能力会被渲染到 developer sections
- 显式提到某个 plugin 时，会把相关技能/MCP/apps 的 grouped guidance 一起注入

这说明 plugin 不只是配置打包，还是**面向模型的能力边界单元**。

也就是说，plugin 同时在三层生效：

1. config / file discovery 层
2. runtime capability summary 层
3. prompt / guidance 层

## app-server 侧没有另起一套 plugin 模型
app-server 的 plugin RPC：
- list
- read
- install
- uninstall

本质上还是包一层 core plugin manager 的结果。

这说明：

- plugin packaging 的真相源仍在 core
- app-server 只是把这套包装暴露给客户端

## 当前边界判断

- plugin manifest：能力声明入口
- plugin manager：能力发现 / 归一化 / summary 构建
- `PluginCapabilitySummary`：runtime 汇总视图
- skills / MCP / apps：plugin 下游三条主要能力线
- `interface.capabilities`：偏展示元数据，不是主 runtime 装配入口

## 一个值得提醒的坑

spec 和代码当前可能有细微不一致：

- 文档容易让人理解成 skills / hooks / mcpServers 都是在“supplement defaults”
- 但代码里 skills 更偏 additive，MCP/apps 更偏 override+fallback

这个点后面如果写 guide，一定要按代码说，不要按文案想当然。

## 设计判断

### 1. plugin 是 Codex 当前最值得关注的 capability packaging unit
这已经比“skills 是能力单元”更上了一层。

### 2. skills 与 MCP 的真正交点，确实主要不是互相调用，而是共享 plugin 装配
这进一步坐实了我们前一批的判断。

### 3. plugin 既服务 runtime，也服务模型理解
这比传统“插件 = 运行时扩展包”更进一步。

## 还值得继续观察的点

1. `interface.capabilities` 后续会不会被真正接入 runtime 行为决策
2. plugin hooks 路径还值得单独补一篇
3. plugin marketplace / install 生命周期还可以继续深拆
