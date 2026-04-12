# STATUS

## Goal

Turn the accumulated Codex Phase 1 source-reading materials into a guidebook-quality, continuously readable architecture narrative without losing the function-level evidence layer.

## Current state

Phase 1 broad architecture scan is effectively complete. The repo has now moved into a **guidebook + deep-dive topics + appendix layer** phase.

Repository currently contains:

- 31 architecture / implementation notes
- 27 fine-grained function-level source-reading notes
- rewritten navigation files (`index.md`, `00-index/...`)
- a guidebook正文 spine in `00-guidebook/` with chapters 00-06
- a topic layer in `00-guidebook/` with deep dives 07-10
- an appendix layer in `00-guidebook/` with:
  - `11-关键函数索引.md`
  - `12-调用链索引.md`
  - `13-open-questions与后续深挖方向.md`
- `LEO_HANDOFF.md` for cross-agent continuation

The repo now has four clearly separated layers:
1. navigation (`index.md`)
2. guidebook正文 (`00-guidebook/` chapters 00-06)
3. deep-dive topics (`00-guidebook/` topics 07-10)
4. evidence (`01-source-notes/`)

Appendices now provide a stable handoff surface for future deep-dive work without forcing another agent to rediscover the most important pivots.

## Completed in this round

### Appendices added
- `00-guidebook/11-关键函数索引.md`
- `00-guidebook/12-调用链索引.md`
- `00-guidebook/13-open-questions与后续深挖方向.md`

### Navigation updates
- root `index.md` now routes readers through three reading layers:
  - main正文
  - topic deep-dives
  - appendices
- reading paths now explicitly point to appendix docs when a reader wants to continue from prose into function/call-chain/open-question space

### Handoff improvements
- “where to continue” is now written into the repo explicitly instead of living only in STATUS/open questions
- key runtime pivots are now indexed by function role, not just by article title
- call-chain reading order is now documented for the major subsystems

## Current high-confidence judgments

- `codex-cli/` is only a distribution shell
- `codex-rs/cli` is the unified command/mode entry layer
- `tui` is a frontend surface increasingly operating over app-server, not the deepest runtime owner
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
- recovery is fundamentally replay-oriented, not ORM-style object reconstruction
- `exec.rs` is the execution primitive layer; `unified_exec` is the sessionful agent-facing execution subsystem
- exec-server is a process/filesystem RPC backend selected through `EnvironmentManager` / `Environment`
- plugin is the shared capability packaging unit above skills / MCP / apps
- plugin install lifecycle is cache-on-disk + config enable + runtime re-derivation
- skills and MCP are parallel capability lines whose strongest visible coupling today is mediated by plugins/config
- apps/connectors is a merged capability surface combining directory metadata, runtime accessibility, and plugin declarations
- `/review` and guardian are separate systems: user review workflow vs approval reviewer infrastructure
- guardian review analytics schema exists and reducer support exists, but runtime emission appears incomplete in this source tree
- realtime and collab are separate systems: realtime conversation vs multi-agent collaboration runtime
- realtime handoff is a bridge back into the ordinary session turn engine, not a fully separate execution world
- `AgentControl` is the rooted thread-tree multi-agent control plane; multi-agent tool handlers are façades over it
- memories is a startup pipeline; agents is a session-scoped multi-agent control plane; external-agent-config is currently migration-oriented infrastructure
- model transport is layered as substrate (`codex-client`) → provider API (`codex-api`) → runtime orchestration (`ModelClient`), while `backend-client` serves a different backend/task API surface
- function-level notes show the repo’s most important runtime pivots are mostly reducer / projection / packaging / reconciliation boundaries, not giant manager objects
- `ThreadHistoryBuilder` is the clearest turn-semantics authority discovered so far
- unified-exec lifecycle is now sufficiently legible end-to-end: request assembly → runtime adaptation → spawn/sessionization → store → output watcher → transcript authority → success/failure end packaging → process-store reconciliation
- the repo should now be read as a layered artifact system: navigation → guidebook正文 → topic deep-dives → evidence, with appendices providing stable drill-down entrypoints

## Next recommended moves

1. do **not** resume broad source scanning unless a real正文 gap appears
2. do a cross-link pass across chapters 01-10 and appendices 11-13
3. do a light consistency pass on terminology across chapters:
   - runtime owner / facade / control plane
   - turn semantic projection / replay / reconstruction
   - topic vs appendix vs evidence
4. if any new work is added, prefer one of these forms only:
   - appendix expansion
   - gap-filling micro-topic
   - narrowly justified function-level note
5. only after the cross-link/consistency pass, decide whether topic 10 should be split

## Open questions

- which request types intentionally skip `ServerRequestResolved` semantics vs which are just not migrated yet
- whether SQLite search/indexing will grow beyond current metadata tables and substring filters
- whether `interface.capabilities` will remain presentation-only or become runtime-significant
- whether app-server will become the dominant long-term embedding surface across all SDKs
- how long TUI will keep a hybrid app-server + legacy-core edge architecture
- whether external-agent-config will remain Claude-first or become a true multi-agent migration layer
- whether exec-server will grow stronger auth/deployment/runtime-orchestration semantics above the current environment contract
- where guardian review analytics is actually emitted in production, if at all
- whether websocket transport will eventually be abstracted lower than `codex-api`, or intentionally remain endpoint-specific
- whether memories phase2 will stay singleton-global or evolve toward sharded/global-partitioned consolidation

## Acceptance bar for current phase

- enough material exists to explain the repo at both architecture and function/state-machine level for the key runtime pivots
- a complete first-pass guidebook正文 spine now exists
- a second-layer topic expansion now covers the most valuable non-spine systems (model transport, backend boundary, guardian/review, realtime/collab, memories/migration)
- an appendix layer now exists for key functions, call chains, and open questions
- enough state has been written into the repo for another agent to continue by cross-linking, polishing, or very targeted gap-filling rather than rediscovering the codebase
- navigation layer, guidebook正文 layer, topic layer, appendix layer, and evidence layer are explicitly separated in the repository
- next work can begin directly from appendix refinement or chapter cross-linking rather than repo re-discovery
