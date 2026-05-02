# WIP PR packets

Draft PR packets that bundle one or more assigned task notes.

Structure:

- one folder per draft PR: `NN-pr-<slug>/`
- one reusable template packet: `pr-template/`

Each draft PR folder should contain:

- `README.md`
  - ship order, dependency, branch, status, upstream target, promotion target, and scope
- `pr-notes.md`
  - working dossier for the bundle: raw material for PR wording, local notes, scope,
    exclusions, validation, open questions, and promotion notes
- `pr-wording.md`
  - draft PR title/body wording
- `tasks/`
  - the assigned task notes included in the draft PR

Task relationship:

- unassigned task notes live in the WIP priority buckets
- once assigned to a draft PR, move the task note into that draft PR packet's `tasks/` folder

How to start a new draft PR packet:

1. Duplicate `pr-template/` to `NN-pr-<slug>/`
2. Assign the next ship-order number in the folder name
3. Fill in `README.md` with ship order, dependency, branch, status, upstream target, promotion target, and scope
4. Move the assigned task notes into `NN-pr-<slug>/tasks/`
5. Use `pr-notes.md` as the working dossier for the draft PR
6. Use `pr-wording.md` for draft PR title/body text

Ordering:

- use the numeric prefix to show the intended ship order for draft PR packets
- record `depends on` in the packet `README.md` when a PR is blocked on another PR or merge
- use both when needed:
  - numeric prefix for queue order
  - `depends on` for actual merge dependency

Promotion tracking:

- use the packet `README.md` to show, at a glance:
  - intended upstream branch
  - current shipping state
  - intended upstream target
  - exact commit or range to promote later
- use `pr-notes.md` to record the mechanical promotion plan:
  - which upstream dependency or merge is being waited on
  - the exact commit or range to cherry-pick
  - any temporary extracted branch/worktree that exists for local validation

PR wording guidance:

- `pr-wording.md` should be the early form of the eventual reviewer-facing PR body, not just a
  scratch note.
- Prefer omission over filler. If a section has no real content, remove it or say so plainly.
- Use the top section that matches the PR type:
  - fix: `Problem` (and `Cause` if useful)
  - feature: `Goal`
  - refactor: `Motivation`
- Then use:
  - `What Changed`
  - `Scope`
  - `For review`
    - `Consequences`
    - `Risks`
    - `Validation`
- Section rules:
  - `What Changed`
    - concrete implementation changes only
    - do not put justification, reviewer commentary, or hypothetical impact here
  - `Consequences`
    - practical consequences of the PR that a reviewer should evaluate
    - do not restate implementation details
    - do not include abstract maintainability benefits unless they have concrete review meaning
  - `Risks`
    - only real, current uncertainties or plausible regressions
    - do not include "this touches an important area" as filler
    - if there is no meaningful risk, say so plainly or remove the section
  - `Validation`
    - exact checks run, plus meaningful caveats or gaps
