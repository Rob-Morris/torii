# Shipped PRs

Archive of shipped upstream PR packets.

Structure:

- one folder per shipped PR: `pr-<number>/`

Each shipped PR folder should contain:

- `README.md`
- `pr-notes.md`
- `pr-wording.md`

It may also contain:

- `tasks/`
  - completed task notes that shipped as part of that PR

Shipping procedure:

1. Do the work from task notes in `.dev.local/wip/`.
2. If a draft PR packet exists under `.dev.local/wip/prs/NN-pr-<slug>/`, move or recreate it under
   `shipped/pr-<number>/` when the PR ships.
3. If the packet was moved, keep its `tasks/` folder as the shipped task record. If it was
   recreated, copy the relevant completed task notes into `shipped/pr-<number>/tasks/`.
4. Keep the shipped packet as the durable PR archive; keep only still-active work in `wip/`.

PR formation guardrails:

- Keep follow-up PRs narrow and explicitly scoped.
- Do not mix new local-only work into the open upstream PR branch.
