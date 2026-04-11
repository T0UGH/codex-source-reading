# STATUS

## Goal

Build Phase 1 source-reading materials for OpenAI Codex before attempting any guidebook-style rewrite.

## Current state

Fourth batch completed. Repository now covers repo map, module boundary map, app-server/core bridge, rollout recovery, exec layering, mcp-server product boundary, plugin-mediated skills/MCP coupling, plus the listener/event projection chain and the split between internal thread state bookkeeping vs outward thread watch state.

## Completed in this round

### New source notes
- listener / event projection chain
- `thread_watch_manager` vs `thread_state_manager` ownership split

### Strengthened judgments
- listener task is the serialized event pump and ordering point for app-server thread delivery
- `bespoke_event_handling` is the real core-event → app-server-protocol projector
- `thread_state_manager` owns subscription/listener/internal-thread state bookkeeping
- `thread_watch_manager` owns outward observable thread status projection

## Current high-confidence judgments

- `codex-cli/` is only a distribution shell
- `codex-rs/cli` is the unified command/mode entry layer
- `tui` and `exec` are front-ends over deeper runtime surfaces
- `core` is the runtime aggregation center
- `ThreadManager` is the runtime creation/ownership entry, not app-server
- `app-server` is a native control-plane facade over core runtime
- listener task is the thread-event pump; `bespoke_event_handling` is the protocol projector
- `thread_state_manager` and `thread_watch_manager` are intentionally split: internal coordination vs outward status projection
- `mcp-server` is a toolified MCP exposure layer, not the same product surface as app-server
- rollout is the durable history truth source; `state_db_bridge` is a thin bridge, not the main restore logic
- `exec.rs` is the execution primitive layer; `unified_exec` is the sessionful agent-facing execution subsystem
- skills and MCP are parallel capability lines whose strongest visible coupling today is mediated by plugins/config

## Next recommended moves

1. inspect `handle_turn_complete` / `handle_turn_interrupted` and how turn materialization is built for app-server clients
2. inspect pending server-request replay / callback / resolve flow in `outgoing_message.rs`
3. go one layer deeper into `codex_rollout` crate instead of only core-side bridges
4. inspect plugin manifest / capability packaging path that affects both skill roots and MCP provenance
5. inspect how much TUI has already converged onto app-server contract vs remaining core-direct paths

## Open questions

- exact turn-completion materialization path for partial/errored/interrupted turns
- exact pending request replay/resolution lifecycle and callback storage semantics
- exact `codex_rollout` physical internals and state-db table semantics
- whether app-server will become the dominant long-term embedding surface across all SDKs
- how explicit plugin packaging declares both skill and MCP capabilities

## Acceptance bar for current batch

- enough material to explain not just module boundaries, but also the app-server event delivery mechanism
- enough evidence to treat listener/state/watch as separate layers in any later guidebook rewrite
- enough state handoff for a later session to continue from turn materialization / pending request replay instead of redoing current listener judgments
