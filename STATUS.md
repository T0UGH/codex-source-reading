# STATUS

## Goal

Build Phase 1 source-reading materials for OpenAI Codex before attempting any guidebook-style rewrite.

## Current state

Sixth batch completed. Phase 1 architecture scanning is now broadly complete at the repo/module/system level. Repository now covers repo map, module boundaries, app-server/core bridge, listener/event projection, turn materialization, pending request replay/resolve, rollout/state_db internals, exec layering, plugin capability packaging and install lifecycle, TUI convergence, mcp-server product boundary, review/guardian chain, remote websocket transport, memories/agents/external-agent-config, and plugin-mediated skills/MCP coupling.

## Completed in this round

### New source notes
- plugin marketplace / install / activation lifecycle
- review / guardian / detached review-thread chain
- remote app-server / websocket transport chain
- memories / agents / external-agent-config system

### Strengthened judgments
- plugin activation is filesystem cache + config enablement + runtime derivation, not a one-shot registry call
- review and guardian are related but distinct: user-facing review workflow vs approval-review infrastructure
- remote app-server is the same app-server contract over websocket, not a different control plane
- memories is a startup two-phase generation pipeline; agents is a session-scoped control plane; external-agent-config is currently Claude-first migration infrastructure

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
- plugin install lifecycle is cache-on-disk + config enable + runtime re-derivation
- skills and MCP are parallel capability lines whose strongest visible coupling today is mediated by plugins/config
- TUI main lifecycle/event path is already largely app-server based, though legacy core helpers remain on the edges
- review and guardian are separate systems: user review workflow vs approval reviewer infrastructure
- remote app-server is transport variation over the same contract, not a parallel server model
- memories is a startup pipeline; agents is a session-scoped multi-agent control plane; external-agent-config is currently Claude-oriented migration infrastructure

## Next recommended moves

1. start converting the current Phase 1 notes into a guidebook-style rewrite
2. add dedicated call-chain diagrams for review / guardian / remote transport / memories
3. add operator-facing notes for plugin marketplace and external-agent-config
4. reorganize the current 24 notes into a chapter-oriented reading order
5. only after that, continue into deeper product/policy corners instead of rescanning the main architecture

## Open questions

- which request types intentionally skip `ServerRequestResolved` semantics vs which are just not migrated yet
- whether SQLite search/indexing will grow beyond current metadata tables and substring filters
- whether `interface.capabilities` will remain presentation-only or become runtime-significant
- whether app-server will become the dominant long-term embedding surface across all SDKs
- how long TUI will keep a hybrid app-server + legacy-core edge architecture
- whether external-agent-config will remain Claude-first or become a true multi-agent migration layer

## Acceptance bar for current batch

- enough material to explain the main repo-level architecture and the major cross-cutting subsystems without reopening earlier boundary questions
- enough evidence to begin a guidebook-style rewrite instead of continuing broad exploratory scanning
- enough handoff state that a later session can shift from discovery mode to synthesis mode with minimal recovery cost
