# T002 — Rollback cache recovery

Status:
- shipped in PR `#428`

Primary outcome:
- rollback now restores cache-sensitive state back to committed storage before retry
- processor model lookup distinguishes missing models from real storage failures via
  `model_optional(...)` / storage-backed lookup
- retry no longer silently skips required upgrade work because cache state remained ahead of
  committed SQLite state

Included recovery paths:
- model cache
- token-registration cache
- ERC balance / total-supply diff

Why it mattered:
- this closes the rollback/cache-divergence bug where rolled-back `ALTER TABLE` work could be
  skipped on retry because in-memory state still reflected the failed chunk

Shipped in:
- `2bf7f03b` `fix(indexer): recover committed cache state after rollback`
