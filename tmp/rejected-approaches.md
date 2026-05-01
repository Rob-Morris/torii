# Rejected Approaches — torii skipped-model-upgrades investigation

Approaches considered during the Pistols skipped-model-upgrades / rollback-cache-poison
investigation that should **not** be treated as normal backlog for the upstream PR.

Primary source:
- `~/Development/Underware/pistols/torii-emulator/docs/shaping/research/torii-skipped-model-upgrades.md`
- see "Appendix C — Alternative and rejected fix approaches"

These are separated from `tmp/out-of-scope.md` so the out-of-scope note contains only
plausible follow-up improvements, not ideas we already decided not to pursue.

## 1. F3 — Bypass cache when reading `prev_schema` (not recommended)

Idea:
- make `UpgradeModelProcessor` bypass cache when reading the previous schema, instead of
  using the normal storage/cache path

Why rejected:
- by itself it does not solve the bug
- `Sql::model()` already prefers cache, so this approach turns into a special-case
  "skip cache just for upgrades" path
- the confirmed runtime fix is cleaner: restore cache state on rollback and keep normal
  storage-backed lookup semantics

Pragmatic status:
- do not pursue as a follow-up cleanup
- if the current fix ever proves insufficient, revisit the broader cache/storage design
  directly rather than reintroducing upgrade-specific bypass logic

## 2. F7 — Generic "ignore unknown enum selector" parser patch (not recommended)

Idea:
- swallow unknown enum discriminants generically so replay can keep moving

Why rejected:
- once torii accepts an unknown enum selector, it no longer knows how many felts to
  consume for that variant payload
- for payload-bearing variants this can desynchronize the decode stream and create
  silent corruption that is harder to debug than a hard failure

Pragmatic status:
- do not pursue
- do not list as a future hardening task
- if a workaround is ever needed, it must be narrow and model-specific, not a generic
  parser behavior change

## 3. F8 — Last-resort local workaround

Idea:
- catch the exact logged deserialization failure locally and drop that historical event
  instead of rolling back the whole chunk

Why not normal backlog:
- this is an emergency forward-progress workaround, not an upstream-quality fix
- it trades correctness in the affected historical stream for indexer liveness
- it should only be considered if an operator needs a temporary local escape hatch before
  the real fix is deployed

Pragmatic status:
- not part of the upstream PR plan
- keep only as an emergency operator workaround option
