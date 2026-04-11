# Codex 源码拆解 21：plugin 安装、市场与生效链

## 这篇看什么

回答：**plugin 从 marketplace 到本地安装，再到 runtime 生效，中间到底经历了什么？**

## 先给结论

plugin 生命周期已经比较清楚：

1. marketplace manifest 是发现入口
2. plugin payload 会被复制到本地 cache 目录
3. config 里再写 `plugins.<id>.enabled = true`
4. runtime 每次按 config + installed cache 重新解析 plugin
5. 生效后影响：
   - skills
   - MCP servers
   - apps
   - prompt/plugin guidance

一句话：

> plugin 不是“装完就注册进内存”，而是“落盘 + 配置启用 + runtime 按需解析”。

## 关键代码路径

- `codex-rs/app-server/src/codex_message_processor.rs`
- `codex-rs/core/src/plugins/manager.rs`
- `codex-rs/core/src/plugins/store.rs`
- `codex-rs/core/src/plugins/remote.rs`
- `codex-rs/core/src/plugins/startup_sync.rs`
- `codex-rs/cli/src/marketplace_cmd.rs`

## marketplace/read/list 路径

app-server 侧核心 RPC：

- `plugin/list`
- `plugin/read`
- `plugin/install`
- `plugin/uninstall`

其中：

### `plugin/list`
- 可选先做 `force_remote_sync`
- 再列本地 marketplace roots
- 还会 warm featured ids / 非 curated cache

### `plugin/read`
- 读取 marketplace entry
- 再补 manifest 里的 skills / apps / MCP server names

也就是说，read/list 并不是薄壳，它已经在帮客户端做一层“可展示视图”的归一化。

## plugin payload 最终落到哪

`PluginStore` 会把安装好的 plugin 放到：

- `plugins/cache/<marketplace>/<plugin>/<version>`

版本来自：
- `.codex-plugin/plugin.json`
- 没有就默认 `local`

并且：
- 写入是原子替换
- active version 通过目录扫描选出
- `local` 会优先赢

这说明 plugin store 本质上是：

> 一个文件系统型 artifact cache，而不是数据库 registry。

## install 链路

本地 install 主链：

1. `resolve_marketplace_plugin(...)`
2. `PluginStore::install(...)`
3. 写 config：`plugins.<plugin_id>.enabled = true`
4. 记录 analytics
5. 清 plugin/skill cache
6. app-server 再触发 MCP refresh / OAuth 等后续动作

这里很关键的一点是：

**仅有 payload 落盘还不算“生效”，还必须有 config enable。**

## uninstall 链路

卸载链路基本相反：

1. 删 cache root
2. 删 config 里的 `plugins.<plugin_id>`
3. 记录 analytics
4. 清 plugin/skill cache

所以 uninstall 是同时清：
- 文件系统 artifact
- config enablement

这也是为什么 runtime 下次解析时会彻底看不到它。

## remote sync 不是全量 plugin registry

remote plugin sync 主要是：
- 从 ChatGPT backend 拉用户远端 plugin 状态
- 再和本地 curated marketplace snapshot reconcile

这有两个重要含义：

### 1. remote sync 主要偏 curated 生态
不是任意第三方 marketplace 的统一云端真相源。

### 2. `enabled=false` 当前更像 uninstall
代码里甚至有 TODO，表示现在对 remote disabled 的处理还偏粗糙。

所以这套 remote sync 不能过度神化，更像是：

> 面向 curated marketplace 的账号态对齐机制。

## startup warmup 在做什么

app-server 启动时会触发 plugin startup tasks：

- curated repo sync
- remote plugin additive sync
- featured plugin cache warm

curated repo 还会优先：
- git clone
- 失败再 zipball
- 再失败才 fallback 到 export archive

这说明 plugin marketplace 并不是每次请求时临时拉，而是有一套本地预热机制。

## 真正“生效”的时刻在哪

不是 install API 返回那一刻。

更准确地说：

- install 让 payload 落盘
- config enable 打开开关
- runtime 下一次 `plugins_for_config(...)` 才真正把它解析进当前能力图

这一步之后，plugin 才能通过：
- `Config::to_mcp_config()`
- skill roots
- plugin summary
- developer/plugin instructions

进入主 runtime。

所以 plugin activation 是一种 **derived activation**，不是 register call。

## hooks 集成的判断

这次看下来，一个重要的负判断也比较稳：

- 没看到 plugin lifecycle 明确接进通用 `codex_hooks` runtime
- plugin 的主要集成点仍然是：
  - MCP config
  - skills
  - apps
  - prompt injection

所以别把“plugin 能带能力”误解成“plugin = hooks runtime extension”。

## 当前边界判断

- marketplace：发现与分发入口
- `PluginStore`：artifact cache
- config enablement：生效开关
- `plugins_for_config(...)`：真正 runtime activation 点
- remote sync：curated 账号态对齐，不是全量云 registry

## 设计判断

### 1. 这是“文件缓存 + 派生生效”架构
优点是简单、可恢复、易 debug。

### 2. plugin 启用/停用状态以 config 为准，不以内存注册表为准
这让重启/跨进程一致性更容易做对。

### 3. startup warmup 说明 plugin 市场已经是正式子系统，不是附带功能

## 还值得继续观察的点

1. plugin disable（不卸载）语义是否会补齐
2. non-curated marketplace cache refresh 还可继续深拆
3. plugin hooks 是否未来会正式接入通用 hooks runtime
