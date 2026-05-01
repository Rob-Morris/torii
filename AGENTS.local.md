# Local agent instructions

This file is for local continuation work only. It is not upstream PR material.

## Purpose

Use this when continuing the skipped-model-upgrades investigation or related follow-up
work from local branches after PR `#428`.

## Branch roles

- `fix/pistols-skipped-model-upgrades`
  - upstream PR branch for `https://github.com/dojoengine/torii/pull/428`
  - keep stable; do not mix new local-only work into it
- `wip/pistols-local-notes`
  - local continuation branch
  - preferred branch for further investigation, note-taking, and follow-up experiments

## Start here

Read these in order before changing scope or preparing another PR:

1. `AGENTS.md`
2. `.dev.local/README.md`
3. `.dev.local/wip/README.md`
4. `.dev.local/research/20260501-torii-skipped-model-upgrades-report.md`
5. `.dev.local/research/20260430-torii-skipped-model-upgrades.md`

Then:

- review the task files in `.dev.local/wip/00-first/` before changing scope
- consult the currently relevant shipped PR packet from `.dev.local/shipped/` if a task note points
  you there
- if expanding scope or proposing a different follow-up, consult:
  - `.dev.local/wip/01-next/`
  - `.dev.local/wip/02-later/`
  - `.dev.local/wip/90-rejected/`

## Local-only exclusions

Do not include these in any upstream PR branch unless there is a deliberate, explicit
decision to upstream them:

- `AGENTS.local.md`
- `CLAUDE.local.md`
- `.dev.local/**`
- local-only diagnostics/logging retained for investigation convenience

## PR process

- Do new work on `wip/pistols-local-notes`, not on the open PR branch.
- When a follow-up is ready, create a fresh PR branch from `origin/main` or another deliberate
  base, then create a draft PR packet under `.dev.local/wip/prs/pr-<slug>/`, move the assigned
  task notes into its `tasks/` folder, and port only the intended code changes.
- Before opening a PR, verify that `git diff --name-only` does not include:
  - `.dev.local/`
  - `AGENTS.local.md`
  - `CLAUDE.local.md`
  - any other local-only bootstrap or notes files

## Notes

- The current upstream fix is intentionally narrow.
- `.dev.local/wip/` holds the current local task queue and follow-up notes.
- `.dev.local/research/` holds research documents.
- `.dev.local/shipped/` holds shipped PR packets.
