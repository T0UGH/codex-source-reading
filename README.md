# codex-source-reading

Source-reading materials for understanding OpenAI Codex from the codebase outward.

## Goal

This repo is for **Phase 1** first: extract high-density source-reading materials, not polished public guidebook prose.

Current priorities:
- establish repository layering and system map
- identify primary runtime entrypoints
- extract stable evidence about config, state, capability surfaces, and safety boundaries

## Repository structure

- `00-index/` — indexes, reading maps, code-path lookup cards
- `01-source-notes/` — dense source-reading notes, one topic per file
- `02-call-chain-drafts/` — rough call-chain and flow drafts
- `03-boundary-judgments/` — module responsibility / non-responsibility judgments

## Current batch

1. repository overview and layering
2. why npm is only a distribution shell
3. CLI entrypoint and command routing
4. entry/distribution code-path index

## Working rule

Prioritize:
1. correct conclusions
2. code-path evidence
3. boundary judgments
4. resumable structure

Do not over-optimize for public readability yet.
