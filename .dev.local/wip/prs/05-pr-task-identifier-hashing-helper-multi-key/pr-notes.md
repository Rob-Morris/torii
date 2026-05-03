# Draft PR notes

- TODO: remove sections that do not apply, add sections if this PR needs them, and prefer omission over filler.

## Branch

- intended upstream branch: `refactor/task-identifier-hashing-helper-multi-key`
- current integration branch: `wip/rob`

## Status

- draft packet created
- code committed on `wip/rob` as `73393d1f`
- targeted validation passed locally
- queued behind `01` through `04` for later promotion

## Ordering

- ship order: `05`
- depends on upstream shipment of `03-pr-task-identifier-hashing-helper`
- queued behind `01` through `04`

## Promotion

- branch from fresh `origin/main` once `03` has shipped upstream
- cherry-pick `73393d1f`
- push `refactor/task-identifier-hashing-helper-multi-key`
- open the upstream PR against `dojoengine/torii:main`

## Context

- follows `T007`, which intentionally introduced only the single-key `(from_address, key)` helper
- this packet closes the remaining duplicated three-input `(from_address, keys[1], keys[2])` shape in the `store_*` processors

## Motivation

- four `store_*` processors still repeat the same inline `DefaultHasher` setup for `(from_address, keys[1], keys[2])`
- that leaves one obvious task-ID hashing pattern outside the shared helper path introduced by `T007`

## What Changed

- add a sibling helper for `(from_address, &[keys...])` task identifiers in `task_manager`
- switch the four duplicated `store_*` task-identifier call sites to use it mechanically
- add an order-sensitivity helper test mirroring the earlier single-key coverage

## Consequences

- the four `store_*` task identifiers now share one helper instead of repeating inline hasher setup
- no intended change to task-ID semantics

## Risks

- no new known runtime risk beyond normal refactor risk

## Scope

- helper coverage for the repeated `store_*` three-input task-ID shape only
- direct mechanical call-site rewrites in the four `store_*` processors

## Exclusions

- no change to `IndexingMode::Latest(...)` hashing
- no change to `event_message.rs` task-ID hashing unless it later proves to fit the same helper cleanly
- no broader task-manager redesign

## Validation

- `cargo test -p torii-processors task_id_from_address_and -- --nocapture`
- `cargo check -p torii-processors -p torii-indexer`

## Open questions

- none at the moment

## Notes

- keep this as a narrow continuation of `T007`, not a generalized hashing redesign
