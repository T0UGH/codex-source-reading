# STATUS

## Goal

Build Phase 1 source-reading materials for OpenAI Codex before attempting any guidebook-style rewrite.

## Current state

Fifth batch completed. Repository now covers repo map, module boundary map, app-server/core bridge, listener/event projection, turn materialization, pending request replay/resolve, rollout/state_db internals, exec layering, plugin capability packaging, TUI convergence, mcp-server product boundary, and plugin-mediated skills/MCP coupling.

## Completed in this round

### New source notes
- turn materialization and pending request lifecycle
- `codex_rollout` and `state_db` internal structure
- plugin capability packaging across skills / MCP / apps
- TUI convergence toward app-server contract

### Strengthened judgments
- turn materialization is an event-accumulation + finalization flow, not a one-shot object build
- pending app-server requests are thread-scoped, replayable, abortable, and partially resolved through listener-command ordering
- rollout JSONL is the canonical session body; SQLite is metadata/index/sidecar state
- plugin is the upper capability packaging unit above skills/MCP/apps
- TUI main execution path is already strongly app-server based, with legacy core helpers still around the edges

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
- plugin is the shared capability packaging unit above skills / MCP / apps
- skills and MCP are parallel capability lines whose strongest visible coupling today is mediated by plugins/config
- TUI main lifecycle/event path is already largely app-server based, though legacy core helpers remain on the edges

## Next recommended moves

1. inspect plugin hooks / marketplace / install lifecycle
2. inspect review / guardian / detached review-thread chain
3. inspect remote app-server transport / websocket protocol path
4. inspect memories / agents / external-agent-config system
5. start converting Phase 1 notes into a guidebook-style rewrite

## Open questions

- which request types intentionally skip `ServerRequestResolved` semantics vs which are just not migrated yet
- whether SQLite search/indexing will grow beyond current metadata tables and substring filters
- whether `interface.capabilities` will remain presentation-only or become runtime-significant
- whether app-server will become the dominant long-term embedding surface across all SDKs
- how long TUI will keep a hybrid app-server + legacy-core edge architecture

## Acceptance bar for current batch

- enough material to explain module boundaries, event delivery, turn materialization, persistence structure, plugin packaging, and TUI convergence at a source-reading level
- enough evidence to begin a guidebook-style rewrite without reopening the repo’s major architectural questions
- enough state handoff for later sessions to continue from remaining specialized subsystems instead of redoing current Phase 1 conclusions
