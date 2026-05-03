# Title

`refactor: type model lookup storage errors`

# Body

```md
## Motivation

- `StorageError` was a boxed dynamic error, so callers could not distinguish a missing model from a real lookup failure without adding a second API.
- That forced the temporary `model_optional()` escape hatch even though model lookup only had one meaningful typed branch.

## What Changed

- Replaced the boxed `StorageError` alias with a typed enum.
- Added `StorageError::ModelNotFound { world_address, selector }` for missing model rows.
- Removed `ReadOnlyStorage::model_optional()`.
- Switched model lookup callers to use `model()` and match the typed not-found error instead.

## Scope

- Includes the typed storage error surface, model lookup semantics, and the direct callers and tests affected by removing `model_optional()`.
- Excludes broader sqlite error taxonomy redesign and unrelated cache error changes.

## For review

### Consequences

- Storage callers can now distinguish a missing model from other lookup failures by matching `StorageError::ModelNotFound { .. }`.
- The additive `model_optional()` lookup path is gone; callers use `model()` and match the error instead.

### Risks

- This changes the public storage API surface and the error shape seen by storage consumers.
- Any downstream code that depended on boxed-error behavior or `model_optional()` will need to adapt to the new explicit model-not-found contract.

### Validation

- `cargo check -p torii-storage -p torii-processors -p torii-sqlite -p torii-indexer`
- `cargo test -p torii-sqlite model_returns_model_not_found_when_model_is_missing -- --nocapture`
- `cargo test -p torii-sqlite model_repopulates_cache_after_database_fallback -- --nocapture`
- `cargo test -p torii-indexer test_rollback_replays_model_upgrade_after_cache_reset -- --nocapture`
```
