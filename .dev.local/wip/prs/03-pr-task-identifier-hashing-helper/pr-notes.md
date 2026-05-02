# Draft PR notes

- TODO: remove sections that do not apply, add sections if this PR needs them, and prefer omission over filler.

## Branch

- intended upstream branch: `refactor/task-identifier-hashing-helper`
- current integration branch: `wip/rob`

## Status

- draft packet created
- code implemented on `wip/rob`
- targeted validation passed locally

## Ordering

- ship order: `03`
- no known upstream dependency on `01` or `02`

## Promotion

- promotion target to be recorded once the code is committed cleanly on `wip/rob`

## Context

- follows the rollback/cache and logging cleanups, but is independent of them for upstream ordering
- focused only on the repeated single-key task-ID hashing shape

## Motivation

- several processors and the rollback regression test repeat the same `DefaultHasher` pattern for `(from_address, selector)`
- a shared helper reduces duplication and drift while keeping task-ID behavior explicit

## What Changed

- add a shared helper for `(from_address, key)` task identifiers next to `TaskId`
- switch matching processor call sites to use that helper mechanically
- switch the rollback regression test helper to the same shared function

## Consequences

- the touched task-identifier and dependency call sites now share one hashing helper instead of repeating inline hasher setup
- no intended change to task-ID semantics

## Risks

- no new known runtime risk beyond normal refactor risk

## Scope

- single-key `(from_address, key)` task-ID helper only
- matching processor task IDs and task dependencies
- matching rollback regression test helper usage

## Exclusions

- no refactor of multi-key or canonical-pair task-ID patterns
- no broader task-manager redesign
- no change to task ordering or dependency semantics

## Validation

- `cargo check -p torii-processors -p torii-indexer`
- `cargo test -p torii-indexer test_rollback_replays_model_upgrade_after_cache_reset -- --nocapture`

## Open questions

- none at the moment

## Notes

- keep this as a mechanical helper extraction, not a redesign of task identity
