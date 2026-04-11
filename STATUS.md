# STATUS

## Goal

Build Phase 1 source-reading materials for OpenAI Codex before attempting any guidebook-style rewrite.

## Current state

Second batch completed. Repository now has an initial repo-level map plus first architecture slices for runtime, config/state, capability integration, safety, and embedding.

## Completed in this round

### New source notes
- TUI vs core boundary
- core as runtime aggregation layer
- config model and state restoration
- MCP / hooks / skills integration layer
- sandbox / execpolicy / approval division
- app-server and SDK embedding architecture

### New drafts
- interactive TUI call-chain draft
- exec call-chain draft

### New boundary artifact
- module boundary judgment v1

## Current high-confidence judgments

- `codex-cli/` is only a distribution shell
- `codex-rs/cli` is the unified command/mode entry layer
- `tui` and `exec` are front-ends over deeper runtime surfaces
- `core` is the runtime aggregation center
- `config` and `state` are intentionally split concerns
- MCP / hooks / skills are separate capability-ingress lines
- sandbox / execpolicy / approval are three different control layers
- app-server is the richer control-plane embedding surface; SDKs currently diverge

## Next recommended moves

1. inspect app-server internal bridge from protocol handlers to `ThreadManager`
2. deepen `rollout` / `state_db_bridge` / session reconstruction chain
3. inspect `exec` vs `core::exec` vs `unified_exec` ownership split
4. inspect `mcp-server` vs `app-server` product boundary
5. inspect `skills` ↔ MCP dependency path

## Open questions

- exact app-server → core bridge implementation path
- exact `rollout` crate/file-layout and materialization details
- whether TS SDK will remain CLI-first or later converge toward app-server
- how complete current hook support is intended to become

## Acceptance bar for current batch

- enough material to support Phase 1 repo map and module boundary map
- enough evidence to begin second-wave deeper notes without reopening repo-level questions
