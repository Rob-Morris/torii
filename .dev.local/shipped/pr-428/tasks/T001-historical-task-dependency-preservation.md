# T001 — Historical task dependency preservation

Status:
- shipped in PR `#428`

Primary outcome:
- historical replay tasks now preserve upgrade dependencies when later same-entity work is merged
  into an existing task
- late prerequisites are retained until the prerequisite task is inserted, instead of being
  dropped

Why it mattered:
- this closes the stale-schema historical replay path where post-upgrade events could run against
  the old cached schema and fail with `PrimitiveError(InvalidEnumSelector)`

Shipped in:
- `749049ef` `fix(task-network): preserve late and merged task dependencies`
