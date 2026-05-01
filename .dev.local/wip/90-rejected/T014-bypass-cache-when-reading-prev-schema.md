# T014 — Bypass cache when reading `prev_schema`

Status:
- queue: `90-rejected`

Summary:
- special-case `UpgradeModelProcessor` to bypass cache when reading the previous schema

Why rejected:
- by itself it does not solve the confirmed bug
- it introduces an upgrade-specific cache bypass instead of fixing rollback-time cache state
- the shipped runtime fix is cleaner: restore cache state on rollback and keep normal
  storage-backed lookup semantics
