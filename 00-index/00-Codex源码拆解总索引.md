# Codex 源码拆解总索引

## 当前目标

先建立 Codex 的系统总图，再沿 4 条主线并行拆：

- 入口主线
- 状态主线
- 能力接入主线
- 执行边界主线

## 已完成产物

### 索引

- [[01-代码路径索引-入口与分发]]

### 第一批

- [[../01-source-notes/2026-04-11-codex源码拆解-01-仓库总貌与分层判断]]
- [[../01-source-notes/2026-04-11-codex源码拆解-02-为什么npm只是分发壳]]
- [[../01-source-notes/2026-04-11-codex源码拆解-03-Codex-CLI总入口与命令分流]]

### 第二批

- [[../01-source-notes/2026-04-11-codex源码拆解-04-TUI与core的边界]]
- [[../01-source-notes/2026-04-11-codex源码拆解-05-core作为runtime聚合核心]]
- [[../01-source-notes/2026-04-11-codex源码拆解-06-config模型与状态恢复]]
- [[../01-source-notes/2026-04-11-codex源码拆解-07-MCP-hooks-skills的接入层]]
- [[../01-source-notes/2026-04-11-codex源码拆解-08-sandbox与execpolicy的分工]]
- [[../01-source-notes/2026-04-11-codex源码拆解-09-app-server与SDK嵌入架构]]

### 第三批

- [[../01-source-notes/2026-04-11-codex源码拆解-10-app-server到ThreadManager的桥]]
- [[../01-source-notes/2026-04-11-codex源码拆解-11-rollout-state_db与会话恢复链]]
- [[../01-source-notes/2026-04-11-codex源码拆解-12-exec与unified_exec的职责分工]]
- [[../01-source-notes/2026-04-11-codex源码拆解-13-mcp-server与app-server的产品边界]]
- [[../01-source-notes/2026-04-11-codex源码拆解-14-skills与MCP的依赖路径]]

### 第四批

- [[../01-source-notes/2026-04-11-codex源码拆解-15-listener与event投影链]]
- [[../01-source-notes/2026-04-11-codex源码拆解-16-thread_watch_manager与thread_state_manager的分工]]

### 第五批

- [[../01-source-notes/2026-04-11-codex源码拆解-17-turn-materialization与pending-request链]]
- [[../01-source-notes/2026-04-11-codex源码拆解-18-codex_rollout与state_db内部结构]]
- [[../01-source-notes/2026-04-11-codex源码拆解-19-plugin能力打包与skills-MCP-apps关系]]
- [[../01-source-notes/2026-04-11-codex源码拆解-20-TUI向app-server收敛的现状]]

### 第六批

- [[../01-source-notes/2026-04-11-codex源码拆解-21-plugin安装市场与生效链]]
- [[../01-source-notes/2026-04-11-codex源码拆解-22-review与guardian链路]]
- [[../01-source-notes/2026-04-11-codex源码拆解-23-remote-app-server与websocket链路]]
- [[../01-source-notes/2026-04-11-codex源码拆解-24-memories-agents与external-agent-config]]

### 第七批

- [[../01-source-notes/2026-04-11-codex源码拆解-25-exec-server体系]]
- [[../01-source-notes/2026-04-11-codex源码拆解-26-rmcp-client与OAuth链路]]
- [[../01-source-notes/2026-04-11-codex源码拆解-27-linux-sandbox与process-hardening实现层]]

### 第八批

- [[../01-source-notes/2026-04-11-codex源码拆解-28-analytics与telemetry归约链]]
- [[../01-source-notes/2026-04-11-codex源码拆解-29-realtime与collab子系统]]
- [[../01-source-notes/2026-04-11-codex源码拆解-30-connectors与apps子系统]]

## 下一批建议

1. 在现有 30 篇基础上开始 guidebook-style 重写
2. 补 dedicated call-chain 图：exec-server / rmcp-client / linux-sandbox / analytics / realtime / collab / connectors
3. 若继续实现深挖，下一批可看 backend-client / codex-client / SSE / provider transport
4. 按主题重组全部笔记，收敛成章节结构而不是时间顺序
5. 后续若再继续，优先补图和章节化，而不是继续横向摊新主题

## 当前高优先级问题

1. TUI 和 core 的边界到底在哪里？
2. interactive 与 exec 是否共享同一 runtime 核心？
3. state crate 为什么被单独抽出？
4. app-server 是 UI 服务层还是 embedding 服务层？
