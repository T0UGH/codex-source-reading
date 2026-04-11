# STATUS

## Goal

Build Phase 1 source-reading materials for OpenAI Codex before attempting any guidebook-style rewrite.

## Current state

Function-level source-reading mode has started on top of the existing architecture scan. Repository now has 31 architecture/implementation notes plus the first 4 fine-grained function notes, covering `ensure_listener_task_running_task(...)`, `apply_bespoke_event_handling(...)`, `reconstruct_history_from_rollout(...)`, and `UnifiedExecHandler::handle(...)`.

## Completed in this round

### New source notes
- function-level note on listener installation / ownership
- function-level note on event projection / side-effect hub
- function-level note on rollout reconstruction
- function-level note on unified exec request assembly

### Strengthened judgments
- some of the most important Codex internals are better explained at the function/state-machine level than at the module level
- the next useful deep-dive mode is no longer “new subsystem coverage”, but “single critical function / local chain” coverage

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
- function-level notes are now being used to explain critical runtime pivots that module-level notes still compress too much

## Next recommended moves

1. continue the function-level series for the most critical runtime pivots
2. likely next targets: `handle_thread_listener_command(...)`, `handle_turn_complete(...)`, `UnifiedExecRuntime::run(...)`, `UnifiedExecProcessManager::exec_command(...)`
3. after another batch or two of function-level notes, regroup everything into a guidebook structure
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

- enough material to explain the repo at both architecture level and selected function/state-machine level
- enough handoff state to continue with a fine-grained Codex source-reading series instead of broad subsystem scanning
- enough evidence to later rewrite the material into a guidebook without losing the most important local runtime mechanics
