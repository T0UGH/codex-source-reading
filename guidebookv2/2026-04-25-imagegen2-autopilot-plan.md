# Codex guidebookv2 imagegen2 autopilot plan

Started: 2026-04-25

## Goal

Use Codex CLI image_generation / imagegen2 to add one explanatory technical diagram to every Codex guidebookv2 article.

## Scope

Article trees:

- `guidebookv2/volume-1` through `guidebookv2/volume-6`
- `docs/guidebookv2/volume-1` through `docs/guidebookv2/volume-6`

Article count:

- volume-1: 3
- volume-2: 8
- volume-3: 5
- volume-4: 5
- volume-5: 5
- volume-6: 5
- total: 31

Existing imagegen2 coverage before this run: 0 / 31.

## Naming convention

Generated images live under both asset trees:

- `guidebookv2/assets/codex-v<volume>-<index>-imagegen2-<slug>.png`
- `docs/guidebookv2/assets/codex-v<volume>-<index>-imagegen2-<slug>.png`

Markdown image references should use:

```md
![...](../assets/<image>.png)
```

## Workflow

Proceed volume by volume:

1. Generate a manifest for one volume.
2. Run one `codex exec --enable image_generation` process per article.
3. Save generated PNGs into `guidebookv2/assets/`.
4. Mirror PNGs into `docs/guidebookv2/assets/`.
5. Insert image blocks into both `guidebookv2` and `docs/guidebookv2` article Markdown.
6. Visual-review the generated volume gallery.
7. Run verification:
   - missing imagegen2 refs across both trees
   - bad image refs across both trees
   - `python3 -m mkdocs build --strict`
   - `git diff --check`
8. Commit, push, and verify GitHub Pages.

## Status

- autopilot: armed for 10 mechanical continuation turns
- volume-1: 3/3 generated, inserted into Markdown, visual review passed, local strict build passed, committed and pushed (`3a88b6b`)
- volume-2: 8/8 generated, inserted into Markdown, visual review passed, local strict build passed, committed and pushed (`893b1e0`)
- volume-3: 5/5 generated, inserted into Markdown, visual review passed, local strict build passed, committed and pushed (`43af0fd`), Pages workflow passed (`24937994296`)
- volume-4: 5/5 generated, inserted into Markdown, visual review passed, local strict build passed, committed and pushed (`6835072`), Pages workflow passed (`24938402682`)
- volume-5: 5/5 generated, inserted into Markdown, visual review passed, local strict build in progress
- volume-6: pending
