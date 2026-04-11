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

## 下一批建议

1. plugin hooks / marketplace / install 生命周期细拆
2. review / guardian / detached review thread 链
3. remote app-server transport / websocket 协议链
4. memories / agents / external-agent-config 体系
5. 在现有 Phase 1 材料基础上开始 guidebook-style 重写

## 当前高优先级问题

1. TUI 和 core 的边界到底在哪里？
2. interactive 与 exec 是否共享同一 runtime 核心？
3. state crate 为什么被单独抽出？
4. app-server 是 UI 服务层还是 embedding 服务层？
