# STATUS

## Goal

Build Phase 1 source-reading materials for OpenAI Codex before attempting any guidebook-style rewrite.

## Current state

Seventh batch completed. Phase 1 broad architecture scan plus one layer of implementation deep-dive is now in place. Repository now covers repo map, module boundaries, app-server/core bridge, listener/event projection, turn materialization, pending request replay/resolve, rollout/state_db internals, exec layering, plugin capability packaging and install lifecycle, TUI convergence, mcp-server product boundary, review/guardian chain, remote websocket transport, memories/agents/external-agent-config, exec-server architecture, rmcp-client/OAuth client stack, linux-sandbox implementation, and plugin-mediated skills/MCP coupling.

## Completed in this round

### New source notes
- exec-server architecture
- rmcp-client / OAuth / MCP client recovery stack
- linux-sandbox and process-hardening implementation layer

### Strengthened judgments
- exec-server is a process/filesystem RPC service behind the environment abstraction, not just a shell helper
- MCP client-side complexity sits in transport/auth/recovery, not merely `call_tool`
- Linux sandbox is now bwrap-first, with seccomp/no_new_privs layered inside and legacy Landlock pushed to fallback/reference status

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

## Next recommended moves

1. start converting the current 27 notes into a guidebook-style rewrite
2. add dedicated call-chain diagrams for exec-server / rmcp-client / linux-sandbox / review / guardian / remote transport / memories
3. if continuing implementation deep-dives, next best topics are analytics / realtime / connectors
4. reorganize the current notes into a chapter-oriented reading order
5. avoid re-scanning repo-level architecture; focus only on specialized subsystems from here

## Open questions

- which request types intentionally skip `ServerRequestResolved` semantics vs which are just not migrated yet
- whether SQLite search/indexing will grow beyond current metadata tables and substring filters
- whether `interface.capabilities` will remain presentation-only or become runtime-significant
- whether app-server will become the dominant long-term embedding surface across all SDKs
- how long TUI will keep a hybrid app-server + legacy-core edge architecture
- whether external-agent-config will remain Claude-first or become a true multi-agent migration layer
- whether exec-server will grow stronger auth/deployment/runtime-orchestration semantics above the current environment contract

## Acceptance bar for current batch

- enough material to explain both the repo-level architecture and a first layer of implementation mechanics for the most important subsystems
- enough evidence to shift future work from exploratory scanning to synthesis / guidebook writing
- enough handoff state that later sessions can go directly into diagrams, restructuring, or a few remaining deep technical subsystems without rediscovering the basics
