# STATUS

## Goal

Build Phase 1 source-reading materials for OpenAI Codex before attempting any guidebook-style rewrite.

## Current state

Function-level source-reading mode is now the dominant workstream. Repository now has 31 architecture/implementation notes plus 27 fine-grained function notes, covering listener installation, listener command dispatch, event projection, running-thread resume composition, pending-request resolution emission, rollout reconstruction, turn completion, turn-summary draining, error-to-failure writing, turn-list assembly, thread/turn status reconciliation, thread-history replay reduction, thread-history event reduction dispatch, active-turn snapshot projection, implicit-turn input routing, unified exec request assembly, unified exec runtime launch adaptation, unified exec spawn/session/storage boundaries, unified exec output chunk slicing, UTF-8 prefix guarding, transcript-first aggregated output resolution, unified exec end-event/output watcher chain, unified exec success-end packaging, unified exec failure-end packaging, and unified exec process-store lifecycle reconciliation.

## Completed in this round

### New source notes
- function-level note on `emit_failed_exec_end_for_unified_exec(...)` as unified-exec failure-end packager
- function-level note on `resolve_aggregated_output(...)` as transcript-first output arbiter
- function-level note on `split_valid_utf8_prefix(...)` as UTF-8-safe chunk boundary guard
- function-level note on `refresh_process_state(...)` as process-store lifecycle reconciler
- function-level note on `handle_user_message(...)` as implicit-turn boundary and input ownership function

### Strengthened judgments
- the final unified-exec end semantics are now basically closed: chunk slicing, transcript authority, success/failure packaging, and store reconciliation all line up
- thread-history correctness depends heavily on tiny boundary helpers like `handle_user_message(...)`, not just on top-level builders
- the repo’s most important local mechanics are now captured enough to stop expanding note count and start restructuring into a guidebook

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
- function-level notes now show the repo’s core runtime pivots are mostly reducer/projection/packaging/reconciliation boundaries, not giant manager objects
- `active_turn_snapshot(...)` is semantically closer to a current-or-last turn projection than a strict active-only getter, so it must be read together with `has_active_turn()`
- thread-history turn slicing follows a consistent rule: explicit turn boundaries win, user-message heuristics backfill old streams, and compaction-only turns get special preservation treatment
- unified-exec treats transcript as the primary truth for final aggregated output, while live delta streaming is explicitly budget-limited and secondary
- `refresh_process_state(...)` confirms the process store is not the truth source; `UnifiedExecProcess` is closer to truth and store entries are reconciled against it lazily

## Next recommended moves

1. stop adding more narrow notes unless a clear gap blocks guidebook writing
2. switch to guidebook restructuring next: cluster existing notes into a readable architecture narrative and a function-level appendix
3. likely first guidebook cuts: runtime layering, app-server/thread-state, unified-exec lifecycle, plugin/capability system, persistence/state model
4. keep future source-note additions exception-only; avoid reopening the repo breadth-first

## Open questions

- which request types intentionally skip `ServerRequestResolved` semantics vs which are just not migrated yet
- whether SQLite search/indexing will grow beyond current metadata tables and substring filters
- whether `interface.capabilities` will remain presentation-only or become runtime-significant
- whether app-server will become the dominant long-term embedding surface across all SDKs
- how long TUI will keep a hybrid app-server + legacy-core edge architecture
- whether external-agent-config will remain Claude-first or become a true multi-agent migration layer
- whether exec-server will grow stronger auth/deployment/runtime-orchestration semantics above the current environment contract
- where guardian review analytics is actually emitted in production, if at all

## Acceptance bar for current batch

- enough material to explain the repo at both architecture level and function/state-machine level for the most important runtime pivots
- enough handoff state to stop broad source scanning and begin guidebook-style restructuring with confidence
- enough evidence to later rewrite the material into a guidebook without losing the key local runtime mechanics and ordering guarantees
