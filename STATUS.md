# STATUS

## Goal

Build Phase 1 source-reading materials for OpenAI Codex before attempting any guidebook-style rewrite.

## Current state

Third batch completed. Repository now has repo map, boundary map v1, two call-chain drafts, and a deeper third-wave slice on app-server bridging, rollout recovery, exec layering, mcp-server product boundary, and plugin-mediated skills/MCP coupling.

## Completed in this round

### New source notes
- app-server → ThreadManager bridge
- rollout / state_db / session reconstruction chain
- exec vs unified_exec responsibility split
- mcp-server vs app-server product boundary
- skills ↔ MCP dependency path (reframed as plugin-mediated coupling)

### Updated artifacts
- total index expanded to three batches
- boundary judgment v1 refined with app-server / mcp-server / plugins / unified_exec / rollout-state_db distinctions

## Current high-confidence judgments

- `codex-cli/` is only a distribution shell
- `codex-rs/cli` is the unified command/mode entry layer
- `tui` and `exec` are front-ends over deeper runtime surfaces
- `core` is the runtime aggregation center
- `ThreadManager` is the runtime creation/ownership entry, not app-server
- `app-server` is a native control-plane facade over core runtime
- `mcp-server` is a toolified MCP exposure layer, not the same product surface as app-server
- rollout is the durable history truth source; `state_db_bridge` is a thin bridge, not the main restore logic
- `exec.rs` is the execution primitive layer; `unified_exec` is the sessionful agent-facing execution subsystem
- skills and MCP are parallel capability lines whose strongest visible coupling today is mediated by plugins/config

## Next recommended moves

1. inspect `ensure_conversation_listener` / listener task / event projection from thread events to app-server events
2. inspect `thread_watch_manager` vs `thread_state_manager` ownership split
3. go one layer deeper into `codex_rollout` crate instead of only core-side bridges
4. inspect plugin manifest / capability packaging path that affects both skill roots and MCP provenance
5. inspect how much TUI has already converged onto app-server contract vs remaining core-direct paths

## Open questions

- exact listener/event bridge internals from `CodexThread` to app-server clients
- exact `codex_rollout` physical internals and state-db table semantics
- whether app-server will become the dominant long-term embedding surface across all SDKs
- how explicit plugin packaging declares both skill and MCP capabilities

## Acceptance bar for current batch

- enough material to support a Phase 1 module map plus first deeper architecture slices
- enough evidence to start a second-wave guidebook rewrite without reopening repo-level boundary questions
- enough state handoff for a later session to continue from listener / rollout internals instead of redoing current judgments
