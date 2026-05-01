# T003 — Rollback regression coverage

Status:
- shipped in PR `#428`

Primary outcome:
- regression coverage now exists for both rollback recovery paths

Coverage added:
- model-cache replay after rollback
- token-registration replay after rollback

Why it mattered:
- this locks in the rollback fix and makes future engine/cache refactors much less likely to
  reintroduce the same silent-skip behavior

Shipped in:
- `2bf7f03b` `fix(indexer): recover committed cache state after rollback`
