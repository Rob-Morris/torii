# Draft PR notes

## Branch

- intended branch: `refactor/centralize-rollback-cache-recovery`
- current working branch: `wip/pistols-local-notes`
- likely base branch: `fix/pistols-skipped-model-upgrades`

## Status

- draft packet created
- code implemented locally, not yet split onto a dedicated PR branch

## Context

- follow-up refactor to the rollback/cache recovery fix currently open as upstream PR `#428`
- intended as a narrow stacked follow-up rather than an expansion of `#428`

## Motivation

- The rollback cache recovery path is currently spelled manually in the engine runtime and in both rollback regressions.
- Centralizing that sequence behind one cache API reduces the chance that future rollback-sensitive cache state is added without recovery coverage.

## What Changed

- Added `Cache::reset_to_committed_storage()` to centralize rollback-sensitive cache recovery.
- Switched the engine rollback path to use that API instead of manually calling the three reset operations.
- Switched both rollback regressions to use the same API.
- Added cache-level coverage for the combined committed-state recovery path.

## Consequences

- Rollback cache recovery is now expressed through one API instead of three separate calls.
- Future rollback-sensitive cache state should be added to one recovery point instead of updating each caller independently.

## Risks

- No new known runtime risk beyond normal refactor risk.

## Scope

- centralize rollback-sensitive cache recovery behind one cache API
- replace the duplicated runtime/test rollback reset sequence with that API
- add direct cache-level coverage for the combined reset path

## Exclusions

- no broader cache redesign
- no behavior changes outside rollback cache recovery semantics
- no logging or CLI follow-up work

## Validation

- `cargo test -p torii-cache -- --nocapture`
- `cargo test -p torii-indexer test_rollback_replays_model_upgrade_after_cache_reset -- --nocapture`
- `KATANA_RUNNER_BIN=/path/to/katana cargo test -p torii-indexer test_rollback_resets_token_registry_for_retry -- --nocapture`

## Open questions

- none at the moment
