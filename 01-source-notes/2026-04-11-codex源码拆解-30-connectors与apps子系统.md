# Codex 源码拆解 30：connectors 与 apps 子系统

## 这篇看什么

回答：**Codex 的 apps/connectors 是怎么发现、鉴权、和 plugins/MCP 结合，并最终展示到 app-server/TUI 的？**

## 先给结论

connectors/apps 这块当前的真实结构是：

1. 目录侧（directory）
   - 知道“有哪些 app/connectors”
2. MCP 侧（codex-apps MCP）
   - 知道“哪些 app 现在真的可用/可调用”
3. plugin 侧
   - 额外声明某些 connector ids
4. app-server/TUI
   - 再把这些信息拼成用户看到的 apps 列表与插件 app 视图

一句话：

> app/connectors 的核心不是单一清单，而是“目录信息 + MCP 可用性 + plugin 元数据”的合并结果。

## 关键代码路径

- `codex-rs/connectors/src/lib.rs`
- `codex-rs/chatgpt/src/connectors.rs`
- `codex-rs/core/src/connectors.rs`
- `codex-rs/core/src/plugins/manager.rs`
- `codex-rs/app-server/src/codex_message_processor.rs`
- `codex-rs/app-server/src/codex_message_processor/apps_list_helpers.rs`
- `codex-rs/app-server/src/codex_message_processor/plugin_app_helpers.rs`
- `codex-rs/tui/src/chatwidget.rs`

## 两个数据源：directory vs accessible

### directory
来自 ChatGPT directory HTTP endpoints。

它知道的是：
- app 名字
- 展示信息
- 目录级元数据

### accessible
来自特殊的 `codex-apps` MCP server。

它知道的是：
- 当前哪些 connector/app 真正可用
- 哪些工具已经暴露出来
- 某些 auth/access 状态

所以 accessible 更接近“运行态”，directory 更接近“目录态”。

## merge 的意义
系统会把这两边 merge：

- directory 提供全量目录元数据
- accessible 标出哪些现在可访问

结果就是：

> 既能展示整个 app 世界，又能知道当前哪些真的能用。

这是很典型的产品层设计。

## plugin 在这里扮演什么角色
plugin 里的 `.app.json` 会声明 connector ids。

这些 ids：
- 进入 plugin capability summary
- 再被 merge 到 app/connectors 视图里

所以 plugin 不是直接让 app“可用”，而是提供：

> 这个 plugin 关心/携带哪些 apps/connectors 的元数据绑定。

真正“现在能不能用”，还得看 codex-apps MCP 是否 ready、是否暴露 tools、是否已经 auth。

## app-server 的角色
app-server 既暴露：
- `app/list`

也在 plugin 相关 RPC 里补：
- `plugin/read` 的 `apps`
- `plugin/install` 的 `apps_needing_auth`

这说明 app-server 在 apps 这块做了两层工作：

1. 直接的 apps list surface
2. plugin 视角下的 app auth gap 诊断

## `apps_needing_auth` 的真实意义
`plugin/install` 之后，app-server 会检查：
- plugin 声明的 app connector
- 当前 accessible connector 集合

如果某个 connector 在 plugin metadata 里有，但当前 accessible 里没有，
就可能被标为：
- `needs_auth`

所以这个字段不是“目录里没有”，而是：

> plugin 希望你用，但当前 runtime 还没真正拿到它。

## auth 的关系
这块还有两条 auth 线：

### app connector auth
- 与 codex-apps MCP readiness / connector accessibility 有关
- 登录 scope 里也会请求 `api.connectors.read` / `invoke`

### plugin MCP OAuth
- plugin 自带 MCP servers 也可能单独触发 OAuth login

这两条线相关，但不是一回事。

所以不要把：
- connector 可访问性
n- plugin MCP server OAuth

混成一套 auth 模型。

## TUI 怎么展示 apps
TUI 里：
- apps popup / `$` mentions
- connectors loaded / refresh
- partial snapshot
- full metadata update

说明 TUI 并不是等所有 app 数据全齐才展示，
而是支持：

> accessible-first partial update，再补 full directory metadata。

这和 app-server 的 incremental `AppListUpdated` 通知是一致的。

## 一个成熟的点：增量更新
当前 apps list 不是单次静态返回。

app-server 可能：
- 先发 accessible-only 的 interim update
- 再发 final 合并结果

TUI 也会配合做部分更新。

这是成熟的 UX 取舍：
- 先快
- 再全

## 当前边界判断

- directory：目录态 app 元数据
- accessible apps：运行态可访问 app 集合
- codex-apps MCP：accessible 状态来源
- plugin `.app.json`：plugin 与 connector 的声明性绑定
- app-server：apps list + plugin-app auth gap 投影层
- TUI：增量 apps/connectors 展示层

## 设计判断

### 1. apps/connectors 是多源合并子系统，不是一个表
这个判断很重要。

### 2. plugin 和 apps 的关系是声明绑定，不是可用性真相源
可用性仍然看 codex-apps MCP/runtime auth。

### 3. 增量更新设计是对的
因为这类目录+运行态混合数据，天然不该等全量都准备好才渲染。

## 还值得继续观察的点

1. `with_codex_apps_mcp(...)` 以及 codex-apps MCP 的 bootstrap 还能继续深挖
2. app mention parsing/rendering 还能补一篇
3. connector/app 在 analytics 与 plugin attribution 的闭环还可以再系统化梳理
