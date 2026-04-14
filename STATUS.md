# STATUS

## Goal

Turn the accumulated Codex Phase 1 source-reading materials into a guidebook-quality, continuously readable architecture narrative without losing the function-level evidence layer.

## Current state

Phase 1 broad architecture scan is effectively complete. The repo has now moved into a **guidebook + deep-dive topics + appendix layer + micro-gap cleanup** phase.

In parallel, the external guidebook delivery has moved one step further: **Codex 新卷二（runtime core）已经在 knowledge-vault 中一次性写完 8 篇正文，并已整批导入飞书**。这意味着当前工作不再是“继续把新卷二写出来”，而是：

- 接受这 8 篇作为当前 runtime-core volume 的已交付版本
- 在仓库侧同步这一状态，避免后续 agent 继续按“卷二待写”推进
- 如需继续，只做验收、轻修、结构同步，或推进后续卷

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

### External runtime-core volume delivery confirmed

File-level verification against `~/workspace/knowledge-vault/Inbox/Research/` confirms that the new runtime-core volume currently has **8 article files present**:

- `2026-04-12-Codex-新卷二-01-一次请求怎么进入-Codex-的-runtime-主线-v1.md`
- `2026-04-12-Codex-新卷二-02-ThreadManager-CodexThread-Codex-怎么接成主工作链-v1.md`
- `2026-04-12-Codex-新卷二-03-当前工作面是怎么被组织出来的-v1.md`
- `2026-04-12-Codex-新卷二-04-thread-和-turn-为什么不是两个平行名词-v1.md`
- `2026-04-12-Codex-新卷二-05-系统怎么判断这一轮要不要调用能力-v1.md`
- `2026-04-12-Codex-新卷二-06-动作结果怎么重新回到当前工作回合-v1.md`
- `2026-04-12-Codex-新卷二-07-一轮工作回合什么时候继续什么时候收口-v1.md`
- `2026-04-12-Codex-新卷二-08-把整条-runtime-core-主线重新压成稳定运行图-v1.md`

Current verified file-level state:

- all 8 files exist
- all 8 files contain frontmatter + matching H1 titles
- all 8 files are still marked `status: draft` in frontmatter
- Feishu delivery is reported complete via a summary doc plus 8 per-article docs

This means the blocking uncertainty is no longer “whether the volume exists”, but only whether later review should promote these drafts into a more final publishing state.

### Volume 3 real status verified

File-level verification against `~/workspace/knowledge-vault/Inbox/Research/` confirms that volume 3 currently has **5 draft article files plus both a writing-cards file and a production-order file**:

- article files present: `卷三-01` through `卷三-05`
- writing cards present: `2026-04-12-Codex-卷三线程状态恢复-README-writing-cards-v1.md`
- production order present: `2026-04-12-Codex-卷三线程状态恢复-README-production-order-v1.md`

Current verified state for volume 3:

- all 5 article files exist
- all 5 article files contain frontmatter + matching H1 titles
- all 5 article files are still marked `status: draft`
- the volume-level planning layer is structurally complete
- no explicit Feishu delivery trace was found in the checked vault files

So volume 3 should currently be treated as: **structurally complete at draft level, but not yet verified as formally delivered**.

### Volume 4 real status verified

File-level verification against `~/workspace/knowledge-vault/Inbox/Research/` now confirms that volume 4 has **5 draft article files plus both a writing-cards file and a production-order file**:

- article files present: `卷四-01` through `卷四-05`
- writing cards present: `2026-04-12-Codex-卷四控制面-README-writing-cards-v1.md`
- production order present: `2026-04-13-Codex-卷四控制面-README-production-order-v1.md`

Current verified state for volume 4:

- all 5 article files exist
- all 5 article files contain frontmatter + matching H1 titles
- all 5 article files are still marked `status: draft`
- the volume-level execution layer is now structurally complete
- no explicit Feishu delivery trace was found in the checked vault files

So volume 4 should currently be treated as: **structurally complete at draft level, but not yet verified as formally delivered**.

### Coach-style readability / boundary pass applied

A targeted explanatory-writing pass has now been applied to the highest-risk boundary chapters:

- volume 3 chapter 03
- volume 3 chapter 04
- volume 3 chapter 05
- volume 4 chapter 02
- volume 4 chapter 03

This pass focused on:

- stronger reader-problem openings
- clearer chapter-to-chapter boundary separation
- lower-resolution mental models before dense mechanism detail
- less accidental thesis overlap between adjacent chapters
- sharper volume-tail closure for volume 3

### Volume 3 acceptance-style closing pass applied

A first acceptance-style closing pass has now been applied to volume 3.

What was done:

- tightened the chapter 03 / 04 / 05 boundary chain
- reduced chapter-opening meta-explanation in the volume tail
- made chapter 05 more explicitly function as the volume-3 closure chapter
- preserved the core volume conclusion: Codex persists not because it stores history, but because runtime can recover a currently continuable working line

Current read on volume 3 after this pass:

- structure is stable
- closure is stronger than before
- files still remain `draft` until a later formal delivery/acceptance decision

### Volume 4 acceptance-style closing pass applied

A first acceptance-style closing pass has now been applied to volume 4.

What was done:

- tightened the chapter 01 / 02 / 05 spine so the volume reads more like one control-plane arc
- reduced leftover setup/meta-explanation in the volume opener
- made chapter 05 more explicitly function as the volume-4 closure chapter
- reinforced the retained volume takeaway: app-server is not a parallel runtime, but the unified control-plane contract exposed above core runtime

Current read on volume 4 after this pass:

- structure is stable
- the opener / mechanism / closure chain is clearer than before
- files still remain `draft` until a later formal delivery/acceptance decision

### Volume 5 orch drafting started

The unified-exec volume has now moved beyond planning-only state.

Verified artifacts now present under `~/workspace/knowledge-vault/Inbox/Research/`:

- `2026-04-13-Codex-卷五统一执行子系统-README-writing-cards-v1.md`
- `2026-04-13-Codex-卷五统一执行子系统-README-production-order-v1.md`
- `2026-04-13-Codex-卷五-01-为什么-exec-rs-和-unified-exec-不是一回事-v1.md`
- `2026-04-13-Codex-卷五-02-UnifiedExecHandler-和-UnifiedExecRuntime-是怎么把动作装成执行会话的-v1.md`
- `2026-04-13-Codex-卷五-03-为什么-approval-sandbox-policy-不是执行外围而是在执行前就进入主链-v1.md`
- `2026-04-13-Codex-卷五-04-输出为什么先进入-transcript-而不是直接变成最终结果-v1.md`
- `2026-04-13-Codex-卷五-05-为什么-process-store-不是最终权威源而-unified-exec-更像-execution-control-plane-v1.md`

Current read on volume 5:

- volume-level planning layer exists
- production-order exists
- full first-pass drafting batch (01-05) now exists
- all current volume-5 files are still `draft`
- the next likely step is not more skeleton work, but an acceptance-style boundary/closure pass for the volume tail

### Volume 5 acceptance-style pass applied

A first acceptance-style pass has now been applied to volume 5.

What was done:

- checked the 01 / 02 / 03 chain for role overlap between primitive exec, session assembly, and execution-precondition control
- checked the 04 / 05 tail for closure strength and repeated explanation
- repaired the volume-5 closure chapter after detecting corrupted/duplicated text in chapter 05
- kept the retained volume takeaway centered on execution sessions rather than command invocation

Current read on volume 5 after this pass:

- structure is stable
- the 01–03 escalation path is clear enough
- the 04 / 05 tail now closes more cleanly around transcript / process store / end event / execution control plane
- files still remain `draft` until a later formal delivery/acceptance decision

### Volume 6 orch drafting started

The review/collab/high-runtime volume has now moved beyond planning-only state.

Verified artifacts now present under `~/workspace/knowledge-vault/Inbox/Research/`:

- `2026-04-13-Codex-卷六审查协作与高级runtime-README-writing-cards-v1.md`
- `2026-04-13-Codex-卷六审查协作与高级runtime-README-production-order-v1.md`
- `2026-04-13-Codex-卷六-01-为什么-review-和-guardian-不是一回事-v1.md`
- `2026-04-13-Codex-卷六-02-guardian-为什么更像审查基础设施而不是普通feature链-v1.md`
- `2026-04-13-Codex-卷六-03-realtime-collab-与-AgentControl-分别是什么层-v1.md`
- `2026-04-13-Codex-卷六-04-为什么-memories-更像启动管线而不是普通-helper-v1.md`
- `2026-04-13-Codex-卷六-05-为什么说-Codex-正在长成更高层-runtime-而不是只多了一些高级功能-v1.md`

Current read on volume 6:

- volume-level planning layer exists
- production-order exists
- full first-pass drafting batch (01-05) now exists
- all current volume-6 files are still `draft`
- the next likely step is not more skeleton work, but an acceptance-style boundary/closure pass for the volume tail

### Volume 6 acceptance-style pass applied

A first acceptance-style pass has now been applied to volume 6.

What was done:

- checked the 01 / 02 boundary so review workflow and guardian infrastructure do not collapse into one topic
- checked the 03 / 04 boundary so collaboration runtime and memories pipeline do not collapse into one topic
- tightened the volume opener and the volume closure so the retained takeaway is clearer
- reinforced the volume-level judgment that Codex is growing higher-order runtime organization, not merely accumulating advanced features

Current read on volume 6 after this pass:

- structure is stable
- the 01 / 02 and 03 / 04 boundaries are clearer than before
- the final chapter now closes more explicitly as a true volume tail
- files still remain `draft` until a later formal delivery/acceptance decision

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

### Cross-link / terminology pass added
- appended or expanded `相关阅读` / `延伸阅读` across appendices and micro-gap docs (`10`–`15` plus `11`/`12`/`13`)
- normalized lingering terminology drift such as `façade` → `facade`, `projection runtime` → clearer layer wording
- updated appendix/open-questions wording so request semantics now align with the post-audit 5+1+1+2 classification

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
- `AgentControl` is the rooted thread-tree multi-agent control plane; multi-agent tool handlers are facades over it
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
2. do **not** continue treating the new runtime-core volume as unwritten; that work is now externally delivered and should be treated as present
3. if more work is needed on volume 2, make it one of these only:
   - review / reader-friction cleanup
   - terminology tightening
   - structure sync between repo notes and vault drafts
   - status promotion from `draft` only after explicit acceptance criteria are met
4. if you still polish terms, focus on:
   - runtime owner / facade / control plane
   - replay / reconstruction / projection
   - topic / appendix / micro-gap / evidence
5. request-shape audit is now basically done for the current enum; if one more targeted pass is worth doing, it should focus on:
   - whether any future/new request shapes are being added outside this 5+1+1+2 classification
   - or whether legacy V1 request paths are actually on a removal/deprecation path
6. after that, prefer polish over expansion

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
