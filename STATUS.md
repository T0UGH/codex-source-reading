# STATUS

## Goal

Build Phase 1 source-reading materials for OpenAI Codex before attempting any guidebook-style rewrite.

## Current state

First batch of repository-level materials has been created.

## Completed in this round

- repository structure established
- top-level reading index created
- entry/distribution code-path index created
- 3 source-reading notes written:
  - repository overview and layering
  - why npm is only a distribution shell
  - CLI entry and command routing

## Next recommended moves

1. write `codex-rs/tui` note: UI layer vs core boundary
2. write `codex-rs/core` note: runtime aggregation and export surface
3. draft interactive TUI call chain
4. draft exec call chain
5. start config/state slice

## Open questions

- exact TUI ↔ app-server relationship
- exact `core` ↔ `exec` division
- how rollout/session metadata and `state` crate divide responsibilities

## Acceptance bar for current batch

- enough evidence to support repo-level layering judgment
- enough evidence to justify not treating `codex-cli/` as the main implementation
- enough evidence to start reading `tui/core/exec` as the next slice
