# STATUS

## Goal

Turn the accumulated Codex Phase 1 source-reading materials into a guidebook-quality, continuously readable architecture narrative without losing the function-level evidence layer.

## Current state

Phase 1 broad architecture scan is effectively complete, and one additional implementation-deep-dive layer is also in place. Repository currently contains:

- 31 architecture / implementation notes
- 27 fine-grained function-level source-reading notes
- a rewritten root `index.md` that now acts as a guidebook-oriented reading map
- an initial `00-guidebook/` layer with:
  - `README.md` (guidebook writing plan)
  - `2026-04-11-Codex-guidebook-章节与架构设计.md` (chapter / structure design)
  - `00-如何阅读这份导读.md` (intro chapter draft)
- `LEO_HANDOFF.md` for direct takeover by Leo or another agent

The repo is no longer in “keep scanning breadth-first” mode. It has entered **guidebook restructuring mode**.

## Completed in this round

### Guidebook-layer artifacts added
- formal guidebook writing plan in `00-guidebook/README.md`
- explicit chapter / architecture design doc in `00-guidebook/2026-04-11-Codex-guidebook-章节与架构设计.md`
- first正文入口稿 `00-guidebook/00-如何阅读这份导读.md`
- rewritten root `index.md` as a reading map rather than a file dump
- `LEO_HANDOFF.md` for cross-agent continuation

### State handoff improvements
- current phase is now explicitly documented as guidebook restructuring, not further broad source scan
- navigation layer /正文层 / evidence layer split is now written into the repo, not just kept in chat
- next recommended moves now bias toward writing guidebook chapters instead of adding more scattered source notes

## Current high-confidence judgments

- `codex-cli/` is only a distribution shell
- `codex-rs/cli` is the unified command/mode entry layer
- `tui` and `exec` are front-ends over deeper runtime surfaces
- `core` is the runtime aggregation center
- `ThreadManager` is the runtime creation/ownership entry, not app-server
- `app-server` is a native control-plane facade over core runtime
- listener task is the thread-event pump; `bespoke_event_handling` is the protocol projector
- `thread_state_manager` and `thread_watch_manager` are intentionally split: internal coordination vs outward status projection
- turn materialization uses active-turn in-memory state plus persisted rollout turns
- pending server requests are stored with thread scope and can be replayed/aborted across reconnect/resume
- `mcp-server` is a toolified MCP exposure layer, not the same product surface as app-server
- rollout is the durable history truth source; `state_db_bridge` is a thin bridge, not the main restore logic
- rollout JSONL stores session body; SQLite stores metadata/index/sidecar state
- `exec.rs` is the execution primitive layer; `unified_exec` is the sessionful agent-facing execution subsystem
- exec-server is a process/filesystem RPC backend selected through `EnvironmentManager` / `Environment`
- plugin is the shared capability packaging unit above skills / MCP / apps
- plugin install lifecycle is cache-on-disk + config enable + runtime re-derivation
- skills and MCP are parallel capability lines whose strongest visible coupling today is mediated by plugins/config
- TUI main lifecycle/event path is already largely app-server based, though legacy core helpers remain on the edges
- review and guardian are separate systems: user review workflow vs approval reviewer infrastructure
- remote app-server is transport variation over the same contract, not a parallel server model
- memories is a startup pipeline; agents is a session-scoped multi-agent control plane; external-agent-config is currently Claude-oriented migration infrastructure
- rmcp-client is the MCP client transport/auth/recovery wrapper; recovery is currently targeted, not universal
- Linux sandbox implementation is bwrap-first; process-hardening is a separate pre-main hardening layer
- analytics is reducer-centered and depends on lifecycle facts plus custom business facts
- realtime and collab are separate subsystems: realtime conversation vs multi-agent collaboration runtime
- connectors/apps is a merged system combining directory metadata, runtime accessibility, and plugin declarations
- model transport is layered as substrate (`codex-client`) → provider API (`codex-api`) → runtime orchestration (`ModelClient`), while `backend-client` serves a different backend/task API surface
- function-level notes show the repo’s most important runtime pivots are mostly reducer / projection / packaging / reconciliation boundaries, not giant manager objects
- `active_turn_snapshot(...)` is semantically closer to a current-or-last turn projection than a strict active-only getter, so it must be read together with `has_active_turn()`
- thread-history turn slicing follows a consistent rule: explicit turn boundaries win, user-message heuristics backfill old streams, and compaction-only turns get special preservation treatment
- `ThreadHistoryBuilder` is now the clearest turn-semantics authority discovered so far
- unified-exec treats transcript as the primary truth for final aggregated output, while live delta streaming is explicitly budget-limited and secondary
- unified-exec lifecycle is now sufficiently legible end-to-end: request assembly → runtime adaptation → spawn/sessionization → store → output watcher → transcript authority → success/failure end packaging → process-store reconciliation
- `refresh_process_state(...)` confirms the process store is not the truth source; `UnifiedExecProcess` is closer to truth and store entries are reconciled against it lazily
- the repo should now be read as a three-layer system of artifacts: navigation (`index.md`), guidebook正文 (`00-guidebook/`), and evidence (`01-source-notes/`)

## Next recommended moves

1. do **not** resume broad source scanning unless a real guidebook gap appears
2. continue guidebook正文 writing next
3. recommended writing order:
   - `00-guidebook/01-系统总图与分层.md`
   - `00-guidebook/05-unified-exec执行子系统.md`
   - `00-guidebook/03-app-server与thread-turn主线.md`
   - `00-guidebook/02-状态持久化与恢复.md`
   - `00-guidebook/04-turn-history语义层.md`
4. after 2-3 main chapters exist, add appendices:
   - key function index
   - call-chain draft index
   - open questions
5. keep future source-note additions exception-only and explicitly justified

## Open questions

- which request types intentionally skip `ServerRequestResolved` semantics vs which are just not migrated yet
- whether SQLite search/indexing will grow beyond current metadata tables and substring filters
- whether `interface.capabilities` will remain presentation-only or become runtime-significant
- whether app-server will become the dominant long-term embedding surface across all SDKs
- how long TUI will keep a hybrid app-server + legacy-core edge architecture
- whether external-agent-config will remain Claude-first or become a true multi-agent migration layer
- whether exec-server will grow stronger auth/deployment/runtime-orchestration semantics above the current environment contract
- where guardian review analytics is actually emitted in production, if at all

## Acceptance bar for current phase

- enough material exists to explain the repo at both architecture and function/state-machine level for the key runtime pivots
- enough state has been written into the repo for Leo or another agent to take over without reconstructing context from chat
- navigation layer, guidebook正文 layer, and evidence layer are now explicitly separated in the repository
- next work can begin directly from guidebook chapter writing rather than repo re-discovery
