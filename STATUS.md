# STATUS

## Goal

Turn the accumulated Codex Phase 1 source-reading materials into a guidebook-quality, continuously readable architecture narrative without losing the function-level evidence layer.

## Current state

Phase 1 broad architecture scan is effectively complete. The repo has now moved beyond “guidebook restructuring only” into a **guidebook + deep-dive topic expansion** phase.

Repository currently contains:

- 31 architecture / implementation notes
- 27 fine-grained function-level source-reading notes
- rewritten navigation files (`index.md`, `00-index/...`)
- a guidebook正文 spine in `00-guidebook/` with:
  - `00-如何阅读这份导读.md`
  - `01-系统总图与分层.md`
  - `02-状态持久化与恢复.md`
  - `03-app-server与thread-turn主线.md`
  - `04-turn-history语义层.md`
  - `05-unified-exec执行子系统.md`
  - `06-capability与高级子系统.md`
- an additional topic layer in `00-guidebook/` with:
  - `07-model-client与provider请求主链.md`
  - `08-codex-client-codex-api与backend-client分层.md`
  - `09-review工作流与guardian审查基础设施.md`
  - `10-realtime-collab与memory迁移专题.md`
- `LEO_HANDOFF.md` for cross-agent continuation

The repo now has both:
- a readable main正文 spine
- a second-layer topic expansion for the most valuable non-spine systems

Future work should bias toward appendices, cross-linking, and selective gap-filling rather than restarting large breadth-first scans.

## Completed in this round

### Topic deep-dives added
- `00-guidebook/07-model-client与provider请求主链.md`
- `00-guidebook/08-codex-client-codex-api与backend-client分层.md`
- `00-guidebook/09-review工作流与guardian审查基础设施.md`
- `00-guidebook/10-realtime-collab与memory迁移专题.md`

### Navigation updates
- root `index.md` now distinguishes:
  - guidebook main spine
  - deep-dive topic layer
  - evidence layer
- reading paths were extended to include model transport/backend boundary and review/realtime/memory topics

### Topic-layer judgments stabilized
- `ModelClient` is a session/turn orchestration layer, not the raw transport layer
- the model request stack is layered as:
  - `core::ModelClient` → `codex-api` → `codex-client`
- `codex-client` is generic HTTP transport substrate
- `codex-api` is model/provider wire adapter
- `backend-client` is a separate business-resource client, not part of inference transport
- `/review` is a user-facing review workflow; guardian is approval reviewer infrastructure
- guardian is fail-closed and routes privileged approvals back through parent-session authority
- guardian uses a trunk + ephemeral review session model, with fork snapshots for context reuse
- realtime and collab are distinct live control planes:
  - realtime = conversation/session live transport plane
  - collab = rooted thread-tree multi-agent control plane
- memories is a startup pipeline with phase1 extract + phase2 global consolidation
- external-agent-config is migration/compatibility infrastructure, not a core runtime spine
- `McpServerConfig` migration appears scaffolded in type/API shape but not fully implemented in behavior

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
- guardian review analytics schema exists and reducer support exists, but runtime emission appears incomplete in this source tree
- realtime and collab are separate systems: realtime conversation vs multi-agent collaboration runtime
- realtime handoff is a bridge back into the ordinary session turn engine, not a fully separate execution world
- `AgentControl` is the rooted thread-tree multi-agent control plane; multi-agent tool handlers are façades over it
- memories is a startup pipeline; agents is a session-scoped multi-agent control plane; external-agent-config is currently migration-oriented infrastructure
- model transport is layered as substrate (`codex-client`) → provider API (`codex-api`) → runtime orchestration (`ModelClient`), while `backend-client` serves a different backend/task API surface
- function-level notes show the repo’s most important runtime pivots are mostly reducer / projection / packaging / reconciliation boundaries, not giant manager objects
- `ThreadHistoryBuilder` is the clearest turn-semantics authority discovered so far
- unified-exec lifecycle is now sufficiently legible end-to-end: request assembly → runtime adaptation → spawn/sessionization → store → output watcher → transcript authority → success/failure end packaging → process-store reconciliation
- the repo should now be read as a three-layer system of artifacts: navigation (`index.md`), guidebook正文 (`00-guidebook/`), and evidence (`01-source-notes/`)

## Next recommended moves

1. do **not** resume broad source scanning unless a real正文 gap appears
2. add appendices next:
   - key function index
   - call-chain index
   - open questions appendix
   - maybe a “where to continue source-reading by subsystem” appendix
3. do a cross-link pass across chapters 01-10 so each chapter explicitly points to the most relevant note/topic/article neighbors
4. optionally split `10-realtime-collab与memory迁移专题` later if that topic layer grows too large, but do not split preemptively
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
- whether websocket transport will eventually be abstracted lower than `codex-api`, or intentionally remain endpoint-specific
- whether memories phase2 will stay singleton-global or evolve toward sharded/global-partitioned consolidation

## Acceptance bar for current phase

- enough material exists to explain the repo at both architecture and function/state-machine level for the key runtime pivots
- a complete first-pass guidebook正文 spine now exists
- a second-layer topic expansion now covers the most valuable non-spine systems (model transport, backend boundary, guardian/review, realtime/collab, memories/migration)
- enough state has been written into the repo for another agent to continue by polishing, cross-linking, and appendix-writing rather than rediscovering the codebase
- navigation layer, guidebook正文 layer, and evidence layer are explicitly separated in the repository
- next work can begin directly from appendix building rather than repo re-discovery
