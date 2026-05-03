# Draft PR notes

- TODO: remove sections that do not apply, add sections if this PR needs them, and prefer omission over filler.

## Branch

- intended upstream branch: `refactor/typed-storageerror-enum`
- current integration branch: `wip/rob`

## Status

- draft packet created
- code ready locally on `wip/rob`
- targeted validation passed locally

## Ordering

- ship order: `06`
- no known upstream dependency on `01` through `05`

## Promotion

- pending final T009 code commit on `wip/rob`
- later flow:
  - branch from fresh `origin/main`
  - cherry-pick the recorded T009 code commit
  - push `refactor/typed-storageerror-enum`
  - open the upstream PR against `dojoengine/torii:main`

## Context

- the skipped-model-upgrades fix needed additive `model_optional()` because boxed `StorageError` was not matchable
- this packet makes missing-model handling explicit at the storage layer instead of encoding it as a parallel optional lookup path

## Motivation

- `StorageError` was a boxed dynamic error, so callers could not distinguish "model not found" from real lookup failures without adding a second API
- that forced the temporary `model_optional()` escape hatch even though model lookup already had one meaningful typed branch

## What Changed

- replace the boxed `StorageError` alias with a typed enum
- add a typed `StorageError::ModelNotFound { world_address, selector }` variant
- remove `ReadOnlyStorage::model_optional()`
- make `ReadOnlyStorage::model()` return the typed not-found error instead
- update sqlite model lookup, processors, storage test utilities, sqlite tests, and the synthetic indexer test to use the typed error path

## Consequences

- storage callers can now distinguish a missing model from other lookup failures by matching `StorageError::ModelNotFound { .. }`
- the additive `model_optional()` lookup path is gone; callers use `model()` and match the error instead

## Risks

- this changes the public storage API surface and the error shape seen by storage consumers
- any downstream code that depended on boxed-error behavior or `model_optional()` will need to adapt to the new explicit model-not-found contract

## Scope

- typed storage error surface
- model lookup semantics
- direct callers and tests affected by removing `model_optional()`

## Exclusions

- no redesign of the broader sqlite error taxonomy beyond what is needed to expose typed model-not-found
- no changes to cache error types
- no follow-on cleanup to remove now-unused helper paths beyond the direct `model_optional()` call sites

## Validation

- `cargo check -p torii-storage -p torii-processors -p torii-sqlite -p torii-indexer`
- `cargo test -p torii-sqlite model_returns_model_not_found_when_model_is_missing -- --nocapture`
- `cargo test -p torii-sqlite model_repopulates_cache_after_database_fallback -- --nocapture`
- `cargo test -p torii-indexer test_rollback_replays_model_upgrade_after_cache_reset -- --nocapture`

## Open questions

- none at the moment

## Notes

- keep this packet focused on typed storage lookup semantics, not a full storage-error taxonomy redesign
