# T010 — Defer cache writes until `storage.execute()`

Status:
- queue: `02-later`

Summary:
- redesign cache mutation so cache state only advances when queued SQL commits

Why this task exists:
- it would remove the rollback/cache-divergence hazard by construction
- it would also remove the smaller same-chunk “future schema” read window

Why later:
- bigger semantic change than the shipped rollback fix
- changes how same-chunk parallel work observes schema state
- needs focused replay validation and design work
