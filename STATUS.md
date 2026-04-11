# STATUS

## Goal

Build Phase 1 source-reading materials for OpenAI Codex before attempting any guidebook-style rewrite.

## Current state

Ninth batch completed. Phase 1 broad architecture scan plus substantial implementation deep-dive are now in place. Repository now covers repo map, module boundaries, app-server/core bridge, listener/event projection, turn materialization, pending request replay/resolve, rollout/state_db internals, exec layering, exec-server architecture, rmcp-client/OAuth client stack, linux-sandbox implementation, analytics/reducer pipeline, plugin capability packaging and install lifecycle, TUI convergence, realtime/collab, connectors/apps, model transport stack, mcp-server product boundary, review/guardian chain, remote websocket transport, memories/agents/external-agent-config, and plugin-mediated skills/MCP coupling.

## Completed in this round

### New source notes
- backend-client / codex-client / codex-api / Responses transport stack

### Strengthened judgments
- the model transport stack is layered as `codex-client` substrate → `codex-api` provider-aware API layer → `core::ModelClient` runtime orchestration, with `backend-client` as a separate backend/task API client line

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

## Next recommended moves

1. start converting the current 31 notes into a guidebook-style rewrite
2. add dedicated call-chain diagrams for exec-server / rmcp-client / linux-sandbox / analytics / realtime / collab / connectors / model transport
3. reorganize the notes into a chapter-oriented reading order
4. add reader-facing navigation/overview docs instead of continuing time-ordered accumulation
5. if any further deep dives are needed, keep them tightly scoped to a single leftover edge subsystem

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

- enough material to explain both repo-level architecture and the main implementation layers for the most important subsystems
- enough evidence that future work should shift from scanning to synthesis, diagrams, restructuring, and guidebook writing
- enough handoff state that later sessions can proceed directly to synthesis without re-discovering the major architectural picture
