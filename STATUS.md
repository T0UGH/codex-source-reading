# STATUS

## Goal

Turn the accumulated Codex Phase 1 source-reading materials into a guidebook-quality, continuously readable architecture narrative without losing the function-level evidence layer.

## Current state

Phase 1 broad architecture scan is effectively complete. The repo has now moved into a **guidebook + deep-dive topics + appendix layer + micro-gap cleanup** phase.

Repository currently contains:

- 31 architecture / implementation notes
- 27 fine-grained function-level source-reading notes
- rewritten navigation files (`index.md`, `00-index/...`)
- a guidebook正文 spine in `00-guidebook/` with chapters 00-06
- a topic layer in `00-guidebook/` with deep dives 07-10
- an appendix layer in `00-guidebook/` with 11-13
- a micro-gap layer in `00-guidebook/` with:
  - `14-ServerRequestResolved覆盖面与未迁移疑点.md`
  - `15-guardian-analytics为何还没真正接上.md`
- `LEO_HANDOFF.md` for cross-agent continuation

The repo now has a usable progression:
1. main正文
2. deep-dive topics
3. appendices
4. targeted micro-gap docs
5. evidence layer

This is enough structure for future work to be selective rather than exploratory.

## Completed in this round

### Micro-gap docs added
- `00-guidebook/14-ServerRequestResolved覆盖面与未迁移疑点.md`
- `00-guidebook/15-guardian-analytics为何还没真正接上.md`

### Gaps stabilized
- callback-map resolution and `ServerRequestResolved` semantics are now explicitly separated
- 5 V2 thread-scoped request types are confirmed to be fully migrated into `ServerRequestResolved`
- threadless/global request (`ChatgptAuthTokensRefresh`) is now explicitly classified as intentionally outside this semantic model
- deprecated V1 requests (`ApplyPatchApproval`, `ExecCommandApproval`) are now explicitly classified as legacy holdouts rather than the highest-priority gap
- `DynamicToolCall` is now narrowed from “clearest suspicious gap” to a more stable read: transport reuses pending request machinery, but product semantics intentionally run through item lifecycle rather than `ServerRequestResolved`
- guardian analytics are now explicitly classified as:
  - schema/client/reducer present
  - runtime emission absent
  - protocol/UI observability present
  - analytics observability still unwired

### Micro-gap follow-up added
- added `03-boundary-judgments/2026-04-12-DynamicToolCall为什么不走ServerRequestResolved.md`
- added `03-boundary-judgments/2026-04-12-app-server-request-shape分类与收口.md`
- narrowed the `DynamicToolCall` question from “likely not migrated yet” to “intentionally item-scoped, transport-reused, but not `ServerRequestResolved`-based semantics”
- request-shape audit now classifies the current enum as: 5 resolved + 1 semantic split + 1 global bridge + 2 legacy holdouts

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
- callback-map resolution and `ServerRequestResolved` are not the same layer of semantics
- 5 V2 thread-scoped request types are now clearly in the `ServerRequestResolved` model:
  - `CommandExecutionRequestApproval`
  - `FileChangeRequestApproval`
  - `ToolRequestUserInput`
  - `McpServerElicitationRequest`
  - `PermissionsRequestApproval`
- `ChatgptAuthTokensRefresh` is intentionally outside the `ServerRequestResolved` model because it is threadless/global
- deprecated V1 requests remain legacy holdouts rather than first-priority migration targets
- `DynamicToolCall` should no longer be described as the clearest `ServerRequestResolved` gap; it is better modeled as a request-transport reuse plus item-lifecycle semantic split
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
- guardian review analytics schema exists and reducer support exists, but runtime emission is not wired in this source tree
- guardian runtime is observable via `EventMsg::GuardianAssessment` and app-server notifications, but not yet via analytics events
- realtime and collab are separate systems: realtime conversation vs multi-agent collaboration runtime
- realtime handoff is a bridge back into the ordinary session turn engine, not a fully separate execution world
- `AgentControl` is the rooted thread-tree multi-agent control plane; multi-agent tool handlers are façades over it
- memories is a startup pipeline; agents is a session-scoped multi-agent control plane; external-agent-config is currently migration-oriented infrastructure
- model transport is layered as substrate (`codex-client`) → provider API (`codex-api`) → runtime orchestration (`ModelClient`), while `backend-client` serves a different backend/task API surface
- `DynamicToolCall` is not best modeled as an un-migrated `ServerRequestResolved` holdout; it reuses thread-scoped request transport/replay/cancel machinery, but its product semantics are item lifecycle (`ItemStarted`/`ItemCompleted` + `DynamicToolCallResponse`), not resolved-notification semantics
- the current `ServerRequest` enum is now legible as a complete set: 5 V2 thread-scoped requests in resolved semantics, 1 item-lifecycle semantic split (`DynamicToolCall`), 1 global bridge request (`ChatgptAuthTokensRefresh`), and 2 deprecated V1 legacy holdouts
- function-level notes show the repo’s most important runtime pivots are mostly reducer / projection / packaging / reconciliation boundaries, not giant manager objects
- `ThreadHistoryBuilder` is the clearest turn-semantics authority discovered so far
- unified-exec lifecycle is now sufficiently legible end-to-end: request assembly → runtime adaptation → spawn/sessionization → store → output watcher → transcript authority → success/failure end packaging → process-store reconciliation
- the repo should now be read as a layered artifact system: navigation → guidebook正文 → topic deep-dives → appendices → micro-gap docs → evidence

## Next recommended moves

1. do **not** resume broad source scanning unless a real正文 gap appears
2. do a cross-link pass across chapters 01-10, appendices 11-13, and micro-gap docs 14-15
3. do a light terminology consistency pass on:
   - runtime owner / facade / control plane
   - replay / reconstruction / projection
   - topic / appendix / micro-gap / evidence
4. request-shape audit is now basically done for the current enum; if one more targeted pass is worth doing, it should focus on:
   - whether any future/new request shapes are being added outside this 5+1+1+2 classification
   - or whether legacy V1 request paths are actually on a removal/deprecation path
5. after that, prefer polish over expansion

## Open questions

- whether future/new request shapes will preserve the current 5+1+1+2 classification, or introduce a genuinely new semantic bucket
- whether deprecated V1 request paths (`ApplyPatchApproval`, `ExecCommandApproval`) are on an actual removal path or will persist as long-tail compatibility seams
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
- at least two concrete micro-gaps have been pulled out of the open-question pool and turned into focused notes with stable answer shapes
- enough state has been written into the repo for another agent to continue by cross-linking, polishing, or very targeted gap-filling rather than rediscovering the codebase
- navigation layer, guidebook正文 layer, topic layer, appendix layer, micro-gap layer, and evidence layer are explicitly separated in the repository
- next work can begin directly from polish or one more targeted micro-gap rather than repo re-discovery
