# T012 — Self-heal on enum mismatch

Status:
- queue: `02-later`

Summary:
- on `InvalidEnumSelector`, refetch schema, repair cache/storage if needed, and retry decode once

Why this task exists:
- it could provide defense-in-depth against future stale-schema failures

Why later:
- not a root fix
- the shipped patch removes the confirmed path that produced the mismatch
- better treated as a backstop than as immediate follow-up work
