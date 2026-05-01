# Title

`fix: prevent skipped model upgrades across task merges and rollbacks`

# Body

```md
## Problem

- On cold replay and chunk retry, Torii could miss additive model upgrades and leave the indexer database in a poisoned state.
- One failure mode was deserialising historical events against stale schema state, producing `PrimitiveError(InvalidEnumSelector)` when a payload used a discriminant that only existed in the upgraded schema.
- The other failure mode was advancing model metadata ahead of the backing SQLite schema, leaving `models.schema` and `/schema` out of sync with the actual table columns.
- That poisoned state caused follow-on write failures (`table ... has no column named ...`), read failures (`no such column`), and retries against stale in-memory state.

## Cause

- Historical replay tasks could lose upgrade dependencies when later same-entity work was merged into an existing task, allowing post-upgrade events to run against the old cached schema.
- After rollback, queued SQL such as `ALTER TABLE ...` could be undone while the in-memory model and token-registration caches remained ahead of committed SQLite state, so retry logic could silently skip required upgrade work.
- Both hazards are present in every torii release since `v1.5.0` (`2025-04-29`).
- They remained latent until cold replay crossed chunks that contained both a schema upgrade and later same-chunk data that required the upgraded schema.

## What Changed

- Preserve dependencies when tasks are merged or registered late.
- Use storage-backed model lookup at processor call sites so namespace-filtered missing models remain skippable while real storage failures surface.
- Reset rollback-sensitive model, token-registration, and ERC diff cache state back to committed storage after rollback.
- Add regression coverage for both rollback recovery paths.

## Scope

- No fetcher changes.
- No parser-behavior widening.
- No broad cache architecture rewrite.
- Only the two confirmed bug classes and direct regression coverage.

## Review-Relevant Side Effects

- Processor model lookup no longer relies on the in-memory model cache alone; on cache miss it now falls back to committed storage and repopulates the cache.
- Processor model lookup no longer treats every storage error as “model missing”; unexpected storage failures now surface instead of being silently skipped.
- Tasks registered late or merged into existing work now retain their prerequisites, which can delay execution until dependencies are satisfied instead of running prematurely.
- Rollback now clears the in-memory ERC balance and total-supply diff before retry, so failed chunks do not carry stale balance deltas forward.
- Rollback now clears cached model state before retry, so model upgrade work may be replayed instead of being skipped under stale in-memory schema state.
- Rollback now rebuilds the token-registration cache from committed storage before retry, so token registration work from a failed chunk may be retried instead of being suppressed by stale in-memory registration marks.

## Validation

- `cargo test -p torii-task-network`
- `cargo test -p torii-cache`
- `cargo test -p torii-sqlite model_optional`
- `cargo check -p torii-cache -p torii-storage -p torii-sqlite -p torii-processors -p torii-indexer`
- `cargo test -p torii-cli --lib`
- `cargo test -p torii-indexer test_rollback_replays_model_upgrade_after_cache_reset -- --nocapture`
- `KATANA_RUNNER_BIN=/path/to/katana cargo test -p torii-indexer test_rollback_resets_token_registry_for_retry -- --nocapture`
```
