# Title

`refactor: centralize rollback cache recovery`

# Body

```md
## Motivation

- The rollback cache recovery path is currently spelled manually in the engine runtime and in both rollback regressions.
- Centralizing that sequence behind one cache API reduces the chance that future rollback-sensitive cache state is added without recovery coverage.

## What Changed

- Added `Cache::reset_to_committed_storage()` to centralize rollback-sensitive cache recovery.
- Switched the engine rollback path to use that API instead of manually calling the three reset operations.
- Switched both rollback regressions to use the same API.
- Added cache-level coverage for the combined committed-state recovery path.

## Scope

- Includes only the rollback cache recovery API, its direct runtime caller, and matching regression/test updates.
- Excludes broader cache redesign and unrelated follow-up cleanup.

## For review

### Consequences

- No intended change to rollback recovery behavior.
- Rollback recovery now has one canonical API across the engine path and rollback regressions instead of repeated reset sequences.
- Future rollback-sensitive cache state now has one recovery point to update instead of multiple call sites.

### Risks

- No new known runtime risk beyond normal refactor risk.

### Validation

- `cargo test -p torii-cache -- --nocapture`
- `cargo test -p torii-indexer test_rollback_replays_model_upgrade_after_cache_reset -- --nocapture`
- `KATANA_RUNNER_BIN=/path/to/katana cargo test -p torii-indexer test_rollback_resets_token_registry_for_retry -- --nocapture`
```
