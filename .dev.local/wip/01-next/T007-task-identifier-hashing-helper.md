# T007 — Shared task identifier hashing helper

Status:
- queue: `01-next`

Summary:
- extract a shared helper for the repeated `(from_address, key)` task-identifier hashing pattern

Why this task exists:
- multiple event processors reimplement the same `DefaultHasher` shape
- the new rollback regression test had to recreate the same pattern
- a shared helper would reduce duplication and drift

Likely shape:
- add a shared helper next to `TaskId` / task-manager code
- rewrite processor call sites to use it mechanically

Why next:
- straightforward refactor
- broad consistency win
- low behavior risk if kept mechanical
