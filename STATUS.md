# STATUS

## Goal

Build Phase 1 source-reading materials for OpenAI Codex before attempting any guidebook-style rewrite.

## Current state

Function-level source-reading mode is now the dominant workstream. Repository now has 31 architecture/implementation notes plus 22 fine-grained function notes, covering listener installation, listener command dispatch, event projection, running-thread resume composition, pending-request resolution emission, rollout reconstruction, turn completion, turn-summary draining, error-to-failure writing, turn-list assembly, thread/turn status reconciliation, thread-history replay reduction, thread-history event reduction dispatch, active-turn snapshot projection, unified exec request assembly, unified exec runtime launch adaptation, unified exec spawn/session/storage boundaries, unified exec output chunk slicing, unified exec end-event/output watcher chain, and unified exec success-end packaging.

## Completed in this round

### New source notes
- function-level note on `ThreadHistoryBuilder::handle_event(...)` as the shared event-reduction entry
- function-level note on `active_turn_snapshot(...)` as current-turn projection boundary
- function-level note on `process_chunk(...)` as transcript/delta slicing boundary
- function-level note on `emit_exec_end_for_unified_exec(...)` as unified-exec success-end packager

### Strengthened judgments
- `ThreadHistoryBuilder` is now clearly the turn-semantics authority; app-server mostly forwards or lightly repairs around it
- unified-exec output path is now legible as receiver bytes → UTF-8-safe chunk slicing → transcript accumulation → end payload packaging
- many “small helpers” in Codex are actually protocol-boundary functions, not just convenience utilities

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
- function-level notes now show the repo’s core runtime pivots are mostly reducer/projection/packaging boundaries, not giant manager objects
- `active_turn_snapshot(...)` is semantically closer to a current-or-last turn projection than a strict active-only getter, so it must be read together with `has_active_turn()`
- unified-exec treats transcript as the primary truth for final aggregated output, while live delta streaming is explicitly budget-limited and secondary

## Next recommended moves

1. do one final high-value function batch only
2. likely next targets: `emit_failed_exec_end_for_unified_exec(...)`, `resolve_aggregated_output(...)`, `split_valid_utf8_prefix(...)`, `refresh_process_state(...)`, `handle_user_message(...)`
3. after that, switch from note accumulation to guidebook restructuring
4. keep future additions tightly scoped; avoid reopening broad repo-level scans

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
- enough handoff state to continue with a fine-grained Codex source-reading series instead of broad subsystem scanning
- enough evidence to later rewrite the material into a guidebook without losing the key local runtime mechanics and ordering guarantees
