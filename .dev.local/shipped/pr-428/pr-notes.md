# PR 428 record

This document records the final local validation and scope for upstream PR `#428`:

- PR URL: `https://github.com/dojoengine/torii/pull/428`
- PR branch: `fix/pistols-skipped-model-upgrades`
- Current state: `open upstream PR awaiting merge`
- Opened: `2026-05-02`

Source documents:

- `.dev.local/research/20260501-torii-skipped-model-upgrades-report.md`
- `.dev.local/research/20260430-torii-skipped-model-upgrades.md`

## Final PR shape

The upstream branch was reduced to two commits:

1. `749049ef` `fix(task-network): preserve late and merged task dependencies`
2. `2bf7f03b` `fix(indexer): recover committed cache state after rollback`

What that final branch includes:

- `TaskManager::add_parallelized_event_with_dependencies` now merges dependencies into an existing
  task instead of only appending events.
- `TaskNetwork` now retains unresolved dependencies and activates them once the prerequisite task is
  inserted.
- processor model lookup now distinguishes `missing model` from real storage failures via
  `model_optional(...)` / storage-backed lookup, instead of swallowing unrelated errors
  under namespace filtering.
- rollback now restores rollback-sensitive cache state back to committed storage before retry:
  - model cache
  - token-registration cache
  - ERC balance / total-supply diff
- regression coverage now exists for both rollback recovery paths:
  - model-cache replay after rollback
  - token-registration replay after rollback

## What was intentionally left out of the PR

- local `.dev.local/` notes and mirrored research
- the separate CLI clap arg-ID fix / test
- investigation-era extra diagnostics in processor logging
- unrelated `torii-indexer-fetcher` or provider-shape cleanup
- broader cache API refactors beyond the confirmed bug fix

## Validation completed on the PR branch

These checks passed on the final shaped branch:

- `git diff --check`
- `cargo test -p torii-task-network`
- `cargo test -p torii-cache`
- `cargo test -p torii-sqlite model_optional`
- `PATH="/opt/homebrew/bin:$PATH" cargo check -p torii-cache -p torii-storage -p torii-sqlite -p torii-processors -p torii-indexer`
- `cargo test -p torii-cli --lib`
- `PATH="$HOME/.asdf/shims:/opt/homebrew/bin:$PATH" cargo test -p torii-indexer test_rollback_replays_model_upgrade_after_cache_reset -- --nocapture`
- `PATH="$HOME/.asdf/shims:/opt/homebrew/bin:$PATH" KATANA_RUNNER_BIN=/Users/robmorris/Development/Underware/katana/target/debug/katana cargo test -p torii-indexer test_rollback_resets_token_registry_for_retry -- --nocapture`

Additional local validation completed during preparation:

- `bash scripts/rust_fmt.sh --fix`
- `PATH="/opt/homebrew/bin:$PATH" bash scripts/clippy.sh`
- local Dojo fixture state was rebuilt with:
  - `cd crates/types-test && PATH="$HOME/.asdf/shims:/opt/homebrew/bin:$PATH" sozo build -P dev`
  - `cd examples/spawn-and-move && PATH="$HOME/.asdf/shims:/opt/homebrew/bin:$PATH" sozo build -P dev`
- targeted rollback/model-optional tests passed:
  - `PATH="$HOME/.asdf/shims:/opt/homebrew/bin:$PATH" cargo test -p torii-cache -p torii-task-network -p torii-processors -- --nocapture`
  - `PATH="$HOME/.asdf/shims:/opt/homebrew/bin:$PATH" KATANA_RUNNER_BIN=/Users/robmorris/Development/Underware/katana/target/debug/katana cargo test -p torii-indexer test_rollback_ -- --nocapture`
  - `PATH="$HOME/.asdf/shims:/opt/homebrew/bin:$PATH" KATANA_RUNNER_BIN=/Users/robmorris/Development/Underware/katana/target/debug/katana cargo test -p torii-cache -p torii-task-network -p torii-sqlite model_optional -- --nocapture`

## Workspace-suite status

Full workspace `nextest` was re-run with:

- `PATH="$HOME/.asdf/shims:/opt/homebrew/bin:$PATH" KATANA_RUNNER_BIN=/Users/robmorris/Development/Underware/katana/target/debug/katana cargo nextest run --all-features --workspace`

It got past the earlier missing-world-state failures, but still stopped on 8
`torii-indexer-fetcher` tests:

- `test_fetch_comprehensive_multi_contract_spam_with_selective_indexing_and_ordering_validation`
- `test_fetch_pending_basic`
- `test_fetch_pending_filters_reverted_transactions`
- `test_fetch_pending_multiple_contracts_comprehensive`
- `test_fetch_pending_multiple_transactions`
- `test_fetch_pending_to_mined_switching_logic`
- `test_fetch_pending_with_cursor_continuation`
- `test_fetch_pending_with_events_comprehensive`

All 8 failed with the same provider-side parse error:

- `Provider(Other(TransportError(Json(Error("data did not match any variant of untagged enum JsonRpcResponse", line: 0, column: 0)))))`

That failure set was reproduced on clean `origin/main` under the current Katana/provider setup, so
it is pre-existing validation noise and out of scope for PR `#428`.

## Replay validation

Replay validation completed against the local pre-critical Pistols Sepolia DB state:

- source DB: `/private/tmp/pistols-torii-repro-patched.VuJ2O7/torii.db`
- confirmed starting state:
  - world head `2262908`
  - `Config|0|0`
  - `PlayerActivityEvent|0|0`
- fresh replay copy:
  - `/private/tmp/pistols-torii-repro-recheck.l0J0eM`
- config:
  - `/Users/robmorris/Development/Underware/pistols/dojo/torii_sepolia_repro.toml`
- run command:
  - `PATH="$HOME/.asdf/shims:/opt/homebrew/bin:$PATH" RUST_LOG=info ./target/debug/torii --config /Users/robmorris/Development/Underware/pistols/dojo/torii_sepolia_repro.toml --db-dir /private/tmp/pistols-torii-repro-recheck.l0J0eM`
- observed replay milestones:
  - crossed `2273149`
  - crossed `2283390`
  - crossed `2303872` before shutdown

Post-run validation:

- `pistols-Config` table includes `realms_address`
- `pistols-PlayerActivityEvent` DDL includes `EnlistedRankedDuelist` in `activity_check`
- `models` flags are now:
  - `Config|1|0`
  - `PlayerActivityEvent|0|1`
- no `InvalidEnumSelector` surfaced in the captured replay logs

Replay-note correction:

- the earlier in-run `count(*) where activity='EnlistedRankedDuelist'` query was against the
  current-state table `[pistols-PlayerActivityEvent]`, not the append-only historical table
- `set_event_message(...)` stores current event-message rows keyed by `poseidon_hash_many(keys)`
  and uses `ON CONFLICT(id) DO UPDATE`, so later values for the same key overwrite the current row
- the durable historical proof is in `event_messages_historical`, which contains `7` persisted
  `PlayerActivityEvent` rows whose JSON payload includes `EnlistedRankedDuelist`
- this was only a validation/debugging interpretation issue in that run; it does not imply an
  additional torii patch change

## Current status

From the perspective of PR `#428`, the shipped PR work is complete:

- the patch was narrowed to the two confirmed bug classes
- the branch was shaped and pushed
- the PR was opened
- the upstream PR remains open and waiting on review / merge
- targeted validation passed
- replay validation passed
- remaining workspace noise is documented as pre-existing and out of scope
