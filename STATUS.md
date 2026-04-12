# STATUS

## Goal

Turn the accumulated Codex Phase 1 source-reading materials into a guidebook-quality, continuously readable architecture narrative without losing the function-level evidence layer.

## Current state

Phase 1 broad architecture scan is effectively complete, and the repo is now firmly in **guidebook restructuring mode** rather than breadth-first source scanning.

Repository currently contains:

- 31 architecture / implementation notes
- 27 fine-grained function-level source-reading notes
- rewritten navigation files (`index.md`, `00-index/...`)
- a guidebook layer in `00-guidebook/` with:
  - `00-如何阅读这份导读.md`
  - `01-系统总图与分层.md`
  - `02-状态持久化与恢复.md`
  - `03-app-server与thread-turn主线.md`
  - `04-turn-history语义层.md`
  - `05-unified-exec执行子系统.md`
  - `06-capability与高级子系统.md`
- `LEO_HANDOFF.md` for cross-agent continuation

The repo now has a readable正文 spine. Future work should bias toward polishing, appendices, and gap-filling rather than creating another large batch of scattered notes.

## Completed in this round

### Guidebook正文 chapters added
- `00-guidebook/01-系统总图与分层.md`
- `00-guidebook/02-状态持久化与恢复.md`
- `00-guidebook/03-app-server与thread-turn主线.md`
- `00-guidebook/04-turn-history语义层.md`
- `00-guidebook/05-unified-exec执行子系统.md`
- `00-guidebook/06-capability与高级子系统.md`

### Navigation updates
- rewrote root `index.md` so it now points readers to the guidebook chapters first
- made the repo entry explicitly guidebook-first and evidence-second

### Chapter-structure adjustment
- the earlier chapter plan that treated capability systems and advanced systems as two separate major chapters was collapsed into one final combined chapter
- current stable six-chapter正文 spine is:
  1. 系统总图与分层
  2. 状态持久化与恢复
  3. app-server 与 thread-turn 主线
  4. turn-history 语义层
  5. unified-exec 执行子系统
  6. capability 与高级子系统

## Current high-confidence judgments

- `codex-cli/` is only a distribution shell
- `codex-rs/cli` is the unified command/mode entry layer
- `tui` is a frontend surface increasingly operating over app-server, not the deepest runtime owner
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
- recovery is fundamentally replay-oriented, not ORM-style object reconstruction
- `exec.rs` is the execution primitive layer; `unified_exec` is the sessionful agent-facing execution subsystem
- exec-server is a process/filesystem RPC backend selected through `EnvironmentManager` / `Environment`
- plugin is the shared capability packaging unit above skills / MCP / apps
- plugin install lifecycle is cache-on-disk + config enable + runtime re-derivation
- skills and MCP are parallel capability lines whose strongest visible coupling today is mediated by plugins/config
- apps/connectors is a merged capability surface combining directory metadata, runtime accessibility, and plugin declarations
- `/review` and guardian are separate systems: user review workflow vs approval reviewer infrastructure
- realtime and collab are separate systems: realtime conversation vs multi-agent collaboration runtime
- memories is a startup pipeline; agents is a session-scoped multi-agent control plane; external-agent-config is currently migration-oriented infrastructure
- model transport is layered as substrate (`codex-client`) → provider API (`codex-api`) → runtime orchestration (`ModelClient`), while `backend-client` serves a different backend/task API surface
- function-level notes show the repo’s most important runtime pivots are mostly reducer / projection / packaging / reconciliation boundaries, not giant manager objects
- `ThreadHistoryBuilder` is the clearest turn-semantics authority discovered so far
- unified-exec lifecycle is now sufficiently legible end-to-end: request assembly → runtime adaptation → spawn/sessionization → store → output watcher → transcript authority → success/failure end packaging → process-store reconciliation
- the repo should now be read as a three-layer system of artifacts: navigation (`index.md`), guidebook正文 (`00-guidebook/`), and evidence (`01-source-notes/`)

## Next recommended moves

1. do **not** resume broad source scanning unless a real guidebook gap appears
2. do a first editorial pass across the six guidebook chapters for tone consistency, repeated judgments, and cross-links
3. add appendices next:
   - key function index
   - call-chain index
   - open questions appendix
4. decide whether Chapter 6 should later be split only after the appendix/polish pass; do not split preemptively
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
- a complete first-pass guidebook正文 spine now exists
- enough state has been written into the repo for another agent to continue by polishing and restructuring rather than rediscovering the codebase
- navigation layer, guidebook正文 layer, and evidence layer are explicitly separated in the repository
- next work can begin directly from guidebook refinement and appendix writing rather than repo re-discovery
