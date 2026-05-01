# WIP PR packets

Draft PR packets that bundle one or more assigned task notes.

Structure:

- one folder per draft PR: `pr-<slug>/`
- one reusable template packet: `pr-template/`

Each draft PR folder should contain:

- `README.md`
  - branch, status, and scope
- `pr-notes.md`
  - working dossier for the bundle: raw material for PR wording, local notes, scope,
    exclusions, validation, and open questions
- `pr-wording.md`
  - draft PR title/body wording
- `tasks/`
  - the assigned task notes included in the draft PR

Task relationship:

- unassigned task notes live in the WIP priority buckets
- once assigned to a draft PR, move the task note into that draft PR packet's `tasks/` folder

How to start a new draft PR packet:

1. Duplicate `pr-template/` to `pr-<slug>/`
2. Fill in `README.md` with branch, status, and scope
3. Move the assigned task notes into `pr-<slug>/tasks/`
4. Use `pr-notes.md` as the working dossier for the draft PR
5. Use `pr-wording.md` for draft PR title/body text

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
