# Title

`refactor: centralize task identifier hashing`

# Body

```md
## Motivation

- Several processor task identifiers and task dependencies repeat the same `DefaultHasher` setup for `(from_address, selector)`.
- The rollback regression test introduced the same shape locally, which makes this duplication easier to drift over time.

## What Changed

- Added `task_id_from_address_and_key()` next to `TaskId` in `task_manager`.
- Switched the matching single-key task-identifier and task-dependency call sites to use that helper mechanically.
- Switched the rollback regression test helper usage to the same function.

## Scope

- Includes only the repeated single-key `(from_address, key)` hashing pattern.
- Excludes multi-key, canonical-pair, and latest-indexing hash shapes.

## For review

### Consequences

- The touched task-identifier and task-dependency paths now share one helper instead of repeating inline hasher setup.
- No intended change to task-ID or dependency semantics.

### Risks

- No new known runtime risk beyond normal refactor risk.

### Validation

- `cargo check -p torii-processors -p torii-indexer`
- `cargo test -p torii-indexer test_rollback_replays_model_upgrade_after_cache_reset -- --nocapture`
```
