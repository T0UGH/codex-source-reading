# LEO HANDOFF

## Goal

接手 `codex-source-reading` 仓库，从 Phase 1 源码拆解/函数级补料，正式转入 guidebook restructuring 阶段。

## Current status

仓库当前已经完成两层核心积累：

1. **架构/实现层笔记**：31 篇
2. **函数级源码共读笔记**：27 篇

此外，guidebook 重组工作已经起步，当前新增了：

- `00-guidebook/README.md`：guidebook 写作方案
- `00-guidebook/2026-04-11-Codex-guidebook-章节与架构设计.md`：章节与架构设计
- `00-guidebook/00-如何阅读这份导读.md`：guidebook 入口页初稿
- 根 `index.md`：已重写为 guidebook-oriented 阅读地图

## What is already stable

### Repo / runtime level
- `codex-cli/` 只是分发壳，不是主体
- `codex-rs/cli` 是统一入口层
- `core` 是 runtime 聚合中心
- `ThreadManager` 比 app-server 更接近 runtime owner
- `app-server` 是控制面 facade，不是另一套平行 runtime
- `mcp-server` 是 toolified 暴露层，不是 app-server 同类物

### Persistence / recovery level
- rollout JSONL 是 session body / durable truth source
- SQLite 更偏 metadata / index / sidecar
- 历史恢复主要靠 replay / rebuild，不是 ORM 式查表重建

### app-server / turn-history level
- listener task 是 thread-event pump
- `bespoke_event_handling` 是协议投影层
- `thread_state_manager` / `thread_watch_manager` 是有意拆分：内部协调 vs outward status projection
- turn-history 是 client-facing semantic projection，不是 event log 直接镜像
- `ThreadHistoryBuilder` 已可视为 turn 语义权威层
- turn slicing 规则已较稳：显式边界优先，旧流靠 user-message heuristic 回补，compaction-only turn 特殊保留

### unified-exec level
- `exec.rs` 是 primitive layer
- `unified_exec` 是 sessionful、agent-facing execution subsystem
- unified-exec 主线已基本闭环：
  - request assembly
  - runtime adaptation
  - spawn/sessionization
  - store
  - output watcher
  - transcript authority
  - success/failure end packaging
  - process-store reconciliation
- transcript 是 final aggregated output 的 primary truth
- live delta streaming 是预算受限的 secondary channel

## What should NOT be done next

### 不要继续做这些
- 不要再 breadth-first 横向扫描仓库
- 不要继续机械增加零散 source notes，除非发现明确缺口阻塞正文
- 不要把 guidebook 写成目录百科或文件清单
- 不要把函数级共读稿直接拼接成正文

## Recommended next moves

### 第一优先级
把 guidebook 正文真正写起来。

建议顺序：
1. `00-guidebook/01-系统总图与分层.md`
2. `00-guidebook/05-unified-exec执行子系统.md`
3. `00-guidebook/03-app-server与thread-turn主线.md`
4. `00-guidebook/02-状态持久化与恢复.md`
5. `00-guidebook/04-turn-history语义层.md`

原因：
- Chapter 1 定总图
- unified-exec / app-server-thread 是当前证据最充分的两条主线
- 持久化与 turn-history 适合在主线建立后再系统重写

### 第二优先级
在正文推进到 2~3 章后，再补：
- appendix-A-关键函数索引
- appendix-B-call-chain草图索引
- appendix-C-开放问题

## Existing guidebook architecture

推荐结构已经在这里落盘：
- `00-guidebook/README.md`
- `00-guidebook/2026-04-11-Codex-guidebook-章节与架构设计.md`

核心原则：
1. 导航层：`index.md`
2. 正文层：`00-guidebook/`
3. 证据层：`01-source-notes/`

## Where to look first

如果 Leo 现在要立刻开工，建议先读：

1. `index.md`
2. `STATUS.md`
3. `00-guidebook/README.md`
4. `00-guidebook/2026-04-11-Codex-guidebook-章节与架构设计.md`
5. `00-guidebook/00-如何阅读这份导读.md`

然后再回看作为证据基础的关键 source notes：
- `01-source-notes/2026-04-11-codex源码拆解-05-core作为runtime聚合核心.md`
- `01-source-notes/2026-04-11-codex源码拆解-10-app-server到ThreadManager的桥.md`
- `01-source-notes/2026-04-11-codex源码拆解-12-exec与unified_exec的职责分工.md`
- `01-source-notes/2026-04-11-codex源码共读-47-build_turns_from_rollout_items是turn-history回放归约器.md`
- `01-source-notes/2026-04-11-codex源码共读-50-ThreadHistoryBuilder-handle_event是turn语义归约总分发器.md`
- `01-source-notes/2026-04-11-codex源码共读-49-spawn_exit_watcher与start_streaming_output是终态发射链.md`
- `01-source-notes/2026-04-11-codex源码共读-53-emit_exec_end_for_unified_exec是统一exec终态封装器.md`
- `01-source-notes/2026-04-11-codex源码共读-57-refresh_process_state是进程存储与真实生命周期的对账器.md`

## Risks / open edges

这些点仍然没有完全收敛，正文里要谨慎表述：
- 哪些 request type 有意跳过 `ServerRequestResolved`，哪些只是尚未迁移
- SQLite search/indexing 后续会不会加强
- `interface.capabilities` 会不会从展示层变成 runtime-significant
- app-server 会不会成为未来更主导的嵌入 surface
- external-agent-config 会不会从 Claude-first migration infra 变成真正通用层
- guardian analytics 在生产里到底有没有稳定发射

## Git / collaboration discipline

这个仓库默认多人/多 agent 可能并发修改。

每次收尾都要遵守：
1. `git status --short`
2. `git pull --rebase --autostash`
3. `git add ...`
4. `git commit -m "..."`
5. `git push`

如果第一次 pull/push 因 SSL / 网络抖动失败，先重试一次，不要误判成权限问题。

## Acceptance bar for Leo takeover

Leo 接手后，第一阶段不该是“继续读更多源码”，而应该是：
- 成功写出可连续阅读的 `01-系统总图与分层.md`
- 让 guidebook 正文层开始替代 source notes 成为主阅读面
- 同时不破坏现有证据仓与索引结构
