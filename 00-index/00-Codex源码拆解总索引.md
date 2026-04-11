# Codex 源码拆解总索引

## 当前目标

先建立 Codex 的系统总图，再沿 4 条主线并行拆：

- 入口主线
- 状态主线
- 能力接入主线
- 执行边界主线

## 第一批产物

### 已完成

- [[01-代码路径索引-入口与分发]]
- [[../01-source-notes/2026-04-11-codex源码拆解-01-仓库总貌与分层判断]]
- [[../01-source-notes/2026-04-11-codex源码拆解-02-为什么npm只是分发壳]]
- [[../01-source-notes/2026-04-11-codex源码拆解-03-Codex-CLI总入口与命令分流]]

## 下一批建议

1. `codex-rs/tui` 主链
2. `codex-rs/config` 模型与层叠加载
3. `codex-rs/state` / rollout / SQLite 状态关系
4. `sandboxing` / `execpolicy` 的分工
5. `codex-mcp` / `mcp-server` / `rmcp-client` 的接入关系

## 当前高优先级问题

1. TUI 和 core 的边界到底在哪里？
2. interactive 与 exec 是否共享同一 runtime 核心？
3. state crate 为什么被单独抽出？
4. app-server 是 UI 服务层还是 embedding 服务层？
