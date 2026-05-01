# T005 — Centralize rollback cache recovery API

Status:
- queue: `00-first`

Summary:
- replace the current explicit rollback reset sequence with a single rollback-recovery API such as
  `Cache::reset_to_committed_storage()`

Why this task exists:
- it is the follow-up most directly connected to the shipped rollback fix
- it reduces the chance that a future cache-backed field is added without rollback recovery
- it keeps engine rollback code and regression tests from drifting apart

Why it was not part of PR `#428`:
- maintainability improvement, not part of the correctness fix
- the explicit rollback sequence in the shipped PR is already short and correct
