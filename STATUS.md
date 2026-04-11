# STATUS

## Goal

Build Phase 1 source-reading materials for OpenAI Codex before attempting any guidebook-style rewrite.

## Current state

Function-level source-reading mode is now firmly underway. Repository now has 31 architecture/implementation notes plus 7 fine-grained function notes, covering `ensure_listener_task_running_task(...)`, `apply_bespoke_event_handling(...)`, `reconstruct_history_from_rollout(...)`, `UnifiedExecHandler::handle(...)`, `handle_thread_listener_command(...)`, `handle_turn_complete(...)`, and `UnifiedExecRuntime::run(...)`.

## Completed in this round

### New source notes
- function-level note on listener command dispatch / ordering
- function-level note on turn completion finalization
- function-level note on unified exec runtime launch adaptation

### Strengthened judgments
- some Codex internals that looked “simple” at module level only become clear after drilling into tiny sequencing helpers
- the most useful next work is now a controlled function-level series over a small number of critical runtime pivots

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
- function-level notes are now revealing the real value of small sequencing functions and finalization helpers inside the broader module architecture

## Next recommended moves

1. continue the function-level series for a small set of high-value runtime pivots
2. likely next targets: `handle_pending_thread_resume_request(...)`, `resolve_pending_server_request(...)`, `handle_error(...)`, `UnifiedExecProcessManager::open_session_with_exec_env(...)`
3. after another small batch, regroup everything into a guidebook structure
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
