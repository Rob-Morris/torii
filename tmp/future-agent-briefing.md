# Torii Upstream PR Briefing

Date: 2026-05-01

## Goal

Produce upstream-ready torii patches for the two confirmed bug classes from the Pistols replay investigation, plus any tightly-related non-breaking hardening needed to avoid shipping another silent-skip variant.

The upstream target is a PR against torii that is ready for review, not another research-only or diagnostics-only branch.

## Primary Sources

- Repo notes:
  - `tmp/production-readiness.md`
  - `tmp/out-of-scope.md`
- Main research source:
  - `/Users/robmorris/Development/Underware/pistols/torii-emulator/docs/shaping/research/torii-skipped-model-upgrades.md`
- Fix-confirmed snapshot captured in system `/tmp`:
  - `/tmp/research_at_fix_confirmed.md`

`tmp/production-readiness.md` has been updated to reflect the live worktree status for
items (1) and (2); use it together with this note.

## What The Research Established

There are two real torii bugs to fix upstream:

1. Historical event task ordering / dependency bug.
   - `EventMessageProcessor` historical work is grouped by `(world, selector, entity_id)`.
   - For the captured Sepolia repro, earlier same-player `PlayerActivityEvent` work created the historical task before the `EventUpgraded(PlayerActivityEvent)` event in the same chunk.
   - Later post-upgrade events were appended to the existing task without inheriting the upgrade dependency.
   - `TaskNetwork` also dropped dependencies whose prerequisite task had not been inserted yet.
   - Result: the `activity = 18` payload at block `2271871` ran against a stale pre-18 schema and threw `PrimitiveError(InvalidEnumSelector { actual_selector: 18 })`.

2. Rollback cache-poison bug.
   - `register_model` mutates `ModelCache` immediately, but SQL changes commit only at chunk end.
   - On chunk failure, `storage.rollback()` discarded `ALTER TABLE` and other queued SQL, while cache state survived.
   - Retry then saw a post-upgrade schema in cache, computed `new_schema.diff(prev_schema) == None`, and skipped the upgrade permanently.
   - Same commit-sensitive shape exists for token registration cache.

The fix-confirmed replay says the combined patch crossed the bad Sepolia window cleanly and landed both `pistols-Config.realms_address` and the `PlayerActivityEvent` enum expansion.

## Current Torii State

### Already committed on top of `origin/main`

Local `main` is ahead of `origin/main` / `v1.8.15` by two WIP commits:

- `84ab46a1 wip`
- `ff031a09 wip: simplify`

Those commits appear to be the accepted fix for bug class (1) plus the first half of bug class (2):

- `crates/processors/src/task_manager.rs`
  - merges newly discovered dependencies into an existing task instead of only appending the event
- `crates/task-network/src/lib.rs`
  - retains late prerequisites in `pending_dependents`
  - activates them when the prerequisite task is inserted
  - includes passing unit tests for both behaviors
- `crates/indexer/engine/src/engine.rs`
  - clears model cache and balances diff on rollback

Notes:

- `event_message.rs` still contains detailed deserialize failure logging that was useful for investigation. Decide whether to keep it as production-grade diagnostics or trim it before upstreaming.
- `84ab46a1` also contains an unrelated one-line `snapshot.version` CLI arg ID change in `crates/cli/src/options.rs`. That should not ride in the eventual upstream fix PR unless separately justified.

### Currently uncommitted in the worktree

The live worktree adds the remaining additive hardening for bug class (2) and the related token-cache issue:

- `crates/storage/src/lib.rs`
  - adds `ReadOnlyStorage::model_optional(...) -> Result<Option<Model>, StorageError>`
- `crates/sqlite/sqlite/src/storage.rs`
  - implements `model_optional` with cache-first lookup and `fetch_optional`
  - keeps `model()` behavior by mapping `None` back to `sqlx::Error::RowNotFound`
- `crates/processors/src/error.rs`
  - adds a typed `Error::ModelNotFound(Felt)`
- 7 processors now use `ctx.storage.model_optional(...)` instead of swallowing any storage error:
  - `event_message.rs`
  - `store_del_record.rs`
  - `store_set_record.rs`
  - `store_update_member.rs`
  - `store_update_record.rs`
  - `upgrade_event.rs`
  - `upgrade_model.rs`
- `crates/cache/src/lib.rs`
  - adds `Cache::reset_token_registry()`
  - stores `Arc<dyn ReadOnlyStorage>` inside `ErcCache`
  - repopulates token-registration state from committed storage on rollback
- `crates/indexer/engine/src/engine.rs`
  - calls `reset_token_registry().await?` in the rollback arm, so rollback repair failure now
    aborts instead of logging and continuing

This uncommitted delta lines up with items (1) and (2) in `tmp/production-readiness.md`.

## Review Notes On The Live Worktree

### Confirmed good

- `git diff --check` is clean.
- `cargo check -p torii-cache -p torii-storage -p torii-sqlite -p torii-processors -p torii-indexer` passed.
- `cargo test -p torii-task-network` passed:
  - 7 tests
  - includes the new late-dependency and dependency-merge cases
- `cargo test -p torii-cache` passed:
  - `reset_token_registry_drops_uncommitted_marks`
  - `clear_models_empties_the_cache`
- `cargo test -p torii-sqlite model_optional` passed:
  - `storage::tests::model_optional_returns_none_when_model_is_missing`
  - `storage::tests::model_optional_repopulates_cache_after_database_fallback`
- `cargo test -p torii-indexer test_rollback_resets_token_registry_for_retry` passed end to end
  when run with the repo-pinned torii toolchain and the explicitly built Katana binary:
  - `PATH="$HOME/.asdf/shims:/opt/homebrew/bin:$PATH" KATANA_RUNNER_BIN=/Users/robmorris/Development/Underware/katana/target/debug/katana cargo test -p torii-indexer test_rollback_resets_token_registry_for_retry -- --nocapture`
  - nuance: do not rely on whichever `scarb` happens to be first on `PATH`; the torii test
    fixture path needs the versions from `torii/.tool-versions`
- `bash scripts/rust_fmt.sh --fix` passed.
- `PATH="/opt/homebrew/bin:$PATH" bash scripts/clippy.sh` passed.
- targeted package-level tests also passed under the repo-pinned torii toolchain:
  - `PATH="$HOME/.asdf/shims:/opt/homebrew/bin:$PATH" cargo test -p torii-cache -p torii-task-network -p torii-processors -- --nocapture`
  - `PATH="$HOME/.asdf/shims:/opt/homebrew/bin:$PATH" KATANA_RUNNER_BIN=/Users/robmorris/Development/Underware/katana/target/debug/katana cargo test -p torii-cache -p torii-task-network -p torii-sqlite model_optional -- --nocapture`

### Things I would challenge before upstreaming

1. There is still no verified deterministic regression test for the model-cache rollback path.
   - Research explicitly recommends a tiny local world / same-chunk upgrade-plus-failure test.
   - The current engine regression now covers the token-registry half of rollback hardening.

2. The two local WIP commits are not yet PR-shaped.
   - The final upstream series should be split into logical units and should drop unrelated changes.

## Suggested Next Steps

1. Add the deterministic model-cache rollback regression described in the research.
2. Re-run validation:
   - `bash scripts/rust_fmt.sh --fix`
   - `PATH="/opt/homebrew/bin:$PATH" bash scripts/clippy.sh`
   - `KATANA_RUNNER_BIN=katana cargo nextest run --all-features --workspace`
   - replay validation from the known Sepolia pre-critical head (`2262908`)
3. Repackage the work into clean upstream commits:
   - task-manager / task-network dependency fix
   - rollback cache repair + `model_optional`
   - token-registry rollback hardening
   - regression tests

## File/State Snapshot

Worktree status during this review:

- modified:
  - `crates/cache/src/lib.rs`
  - `crates/indexer/engine/src/engine.rs`
  - `crates/processors/src/error.rs`
  - `crates/processors/src/processors/event_message.rs`
  - `crates/processors/src/processors/store_del_record.rs`
  - `crates/processors/src/processors/store_set_record.rs`
  - `crates/processors/src/processors/store_update_member.rs`
  - `crates/processors/src/processors/store_update_record.rs`
  - `crates/processors/src/processors/upgrade_event.rs`
  - `crates/processors/src/processors/upgrade_model.rs`
  - `crates/processors/src/task_manager.rs`
  - `crates/sqlite/sqlite/src/storage.rs`
  - `crates/storage/src/lib.rs`
- untracked:
  - `tmp/`

This briefing was written after reviewing:

- the repo-local `tmp/` docs
- the current worktree diff
- the research document under `Underware/pistols/.../torii-skipped-model-upgrades.md`
- the fix-confirmed `/tmp/research_at_fix_confirmed.md`
