# T010 — Defer cache writes until `storage.execute()`

Status:
- queue: `01-next`

Summary:
- redesign cache mutation so cache state only advances when queued SQL commits

Why this task exists:
- it would remove the rollback/cache-divergence hazard by construction
- it would also remove the smaller same-chunk “future schema” read window

Why next:
- strongest remaining architectural follow-up to the rollback/cache-divergence fix
- removes the confirmed hazard by construction instead of recovering from it after rollback
- still needs focused design and replay validation before it is ready to implement directly
