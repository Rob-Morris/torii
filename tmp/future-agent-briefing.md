# Torii Upstream PR Briefing

Date: 2026-05-01

## Goal

Produce upstream-ready torii patches for the two confirmed bug classes from the Pistols replay investigation, plus any tightly-related non-breaking hardening needed to avoid shipping another silent-skip variant.

The upstream target is a PR against torii that is ready for review, not another research-only or diagnostics-only branch.

Keep the patch narrow:
- fix the two confirmed bug classes
- keep tightly-related non-breaking hardening that directly supports those fixes
- do not broaden into unrelated fetcher/provider cleanup or general torii refactors just because
  they showed up during local validation

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

Local `main` is ahead of `origin/main` / `v1.8.15` by three WIP commits:

- `84ab46a1 wip`
- `ff031a09 wip: simplify`
- `9a0a91a3 wip: harden rollback cache recovery`

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

The live worktree now adds the follow-up engine regression for the model-cache rollback path:

- `crates/indexer/engine/src/test.rs`
  - adds `test_rollback_replays_model_upgrade_after_cache_reset`
  - uses a synthetic world-model upgrade processor plus a one-shot failing processor to prove:
    - first pass mutates cache and queues `register_model`
    - rollback drops the SQL but leaves poisoned cache state
    - `clear_models()` restores retry behavior by forcing `model_optional()` to repopulate from
      committed sqlite
    - second pass replays the schema change and lands the new column

Everything else from items (1) and (2) is already captured in `9a0a91a3`.

The repo-local handoff notes were also updated after the full validation pass:

- `tmp/production-readiness.md`
- `tmp/future-agent-briefing.md`

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
- `cargo test -p torii-indexer test_rollback_replays_model_upgrade_after_cache_reset` passed:
  - `PATH="$HOME/.asdf/shims:/opt/homebrew/bin:$PATH" cargo test -p torii-indexer test_rollback_replays_model_upgrade_after_cache_reset -- --nocapture`
- both rollback regressions pass together:
  - `PATH="$HOME/.asdf/shims:/opt/homebrew/bin:$PATH" KATANA_RUNNER_BIN=/Users/robmorris/Development/Underware/katana/target/debug/katana cargo test -p torii-indexer test_rollback_ -- --nocapture`
- `bash scripts/rust_fmt.sh --fix` passed.
- `PATH="/opt/homebrew/bin:$PATH" bash scripts/clippy.sh` passed.
- targeted package-level tests also passed under the repo-pinned torii toolchain:
  - `PATH="$HOME/.asdf/shims:/opt/homebrew/bin:$PATH" cargo test -p torii-cache -p torii-task-network -p torii-processors -- --nocapture`
  - `PATH="$HOME/.asdf/shims:/opt/homebrew/bin:$PATH" KATANA_RUNNER_BIN=/Users/robmorris/Development/Underware/katana/target/debug/katana cargo test -p torii-cache -p torii-task-network -p torii-sqlite model_optional -- --nocapture`
- targeted `torii-indexer` test lint also passed:
  - `PATH="/opt/homebrew/bin:$PATH" cargo +nightly-2025-05-01 clippy -p torii-indexer --tests -- -D warnings -D future-incompatible -D nonstandard-style -D rust-2018-idioms -D unused -D missing-debug-implementations -A clippy::uninlined_format_args`
- full workspace `nextest` got past the earlier local Dojo fixture failures after rebuilding:
  - `cd crates/types-test && PATH="$HOME/.asdf/shims:/opt/homebrew/bin:$PATH" sozo build -P dev`
  - `cd examples/spawn-and-move && PATH="$HOME/.asdf/shims:/opt/homebrew/bin:$PATH" sozo build -P dev`
  - rerun command:
    `PATH="$HOME/.asdf/shims:/opt/homebrew/bin:$PATH" KATANA_RUNNER_BIN=/Users/robmorris/Development/Underware/katana/target/debug/katana cargo nextest run --all-features --workspace`
  - the rollback regressions, GraphQL tests, and broader workspace surface passed through that run
  - the remaining failures were confined to 8 `torii-indexer-fetcher` pending/preconfirmed tests,
    all failing with the same provider parse error:
    `Provider(Other(TransportError(Json(Error("data did not match any variant of untagged enum JsonRpcResponse", line: 0, column: 0)))))`
- replay validation from the research was re-run successfully against the cleaned-up patch shape:
  - source pre-critical DB:
    `/private/tmp/pistols-torii-repro-patched.VuJ2O7/torii.db`
  - confirmed pre-run state:
    - world head `2262908`
    - `Config|0|0`
    - `PlayerActivityEvent|0|0`
  - copied replay DB:
    `/private/tmp/pistols-torii-repro-recheck.l0J0eM`
  - config:
    `/Users/robmorris/Development/Underware/pistols/dojo/torii_sepolia_repro.toml`
  - run command:
    `PATH="$HOME/.asdf/shims:/opt/homebrew/bin:$PATH" RUST_LOG=info ./target/debug/torii --config /Users/robmorris/Development/Underware/pistols/dojo/torii_sepolia_repro.toml --db-dir /private/tmp/pistols-torii-repro-recheck.l0J0eM`
  - observed replay milestones:
    - crossed `2273149`
    - crossed `2283390`
    - crossed `2303872`
  - post-run state:
    - `pistols-Config` gained `realms_address`
    - `pistols-PlayerActivityEvent` DDL now includes `EnlistedRankedDuelist`
    - `models` rows now report:
      - `Config|1|0`
      - `PlayerActivityEvent|0|1`
    - no `InvalidEnumSelector` appeared in the captured replay logs
  - discrepancy resolved:
    - the temporary `2`-row `EnlistedRankedDuelist` count came from the current-state
      `[pistols-PlayerActivityEvent]` table
    - current event-message rows are keyed by `poseidon_hash_many(keys)` and updated in place via
      `ON CONFLICT(id) DO UPDATE`, so later events for the same player/key can replace the
      current `activity`
    - by the end of replay, those current rows had advanced to `ChallengeCreated`, so the final
      current table correctly showed `0` enlisted rows
    - the append-only proof is in `event_messages_historical`, which now contains `7`
      persisted `PlayerActivityEvent` rows with `EnlistedRankedDuelist` in their JSON payload
    - this was only a validation-surface misunderstanding during the replay review, not a torii
      runtime issue and not a reason to change the patch

### Things I would challenge before upstreaming

1. The three local WIP commits are not yet PR-shaped.
   - The final upstream series should be split into logical units and should drop unrelated changes.

2. The new model-cache regression is synthetic rather than built from a real `ModelUpgraded`
   world event.
   - It does exercise the real engine/storage/cache rollback path and directly proves the cache
     poison failure mode that motivated the fix.
   - If extra realism is desired before upstreaming, replace it later with a fixture-driven
     world upgrade replay; that is confidence-building, not required to keep deterministic
     regression coverage.

3. Full workspace validation is not green yet, but the remaining failures are no longer in the
   rollback patch surface.
   - The 8 failures are all under `crates/indexer/fetcher/src/test.rs`.
   - They all share the same `JsonRpcResponse` parse failure against the current Katana/provider
     setup.
   - They reproduce on clean `origin/main`, so treat them as pre-existing validation noise for
     this patch series rather than evidence against the rollback fix itself.
   - Do not expand this PR into `crates/indexer/fetcher` work.

4. Small cleanup pass result:
   - keep the one-line `snapshot.version` arg ID change in `crates/cli/src/options.rs`
     because it avoids a clap arg-ID collision with the top-level built-in `version` flag
   - a new CLI test now guards that:
     `args::test::test_clap_definition_is_valid`
   - `TaskNetwork::contains_key` was unused and has been removed

## Suggested Next Steps

1. Repackage the work into clean upstream commits now that the replay validation has been rerun.
   - task-manager / task-network dependency fix
   - rollback cache repair + `model_optional`
   - token-registry rollback hardening
   - regression tests
2. Keep the PR scoped.
   - Leave `crates/indexer/fetcher` alone for this fix series.
   - If validation notes mention the 8 failing fetcher tests, explicitly say they reproduce on
     clean `origin/main` and are out of scope for this patch.
3. Do not spend more time on the `EnlistedRankedDuelist` row-presence discrepancy unless someone
   reopens the question; it was explained by current-state overwrite semantics and the historical
   proof is already present in `event_messages_historical`.

## File/State Snapshot

Worktree status during this review:

- modified:
  - `crates/cli/src/args.rs`
  - `crates/cli/src/options.rs`
  - `crates/task-network/src/lib.rs`
  - `tmp/future-agent-briefing.md`
  - `tmp/production-readiness.md`

This briefing was written after reviewing:

- the repo-local `tmp/` docs
- the current worktree diff
- the research document under `Underware/pistols/.../torii-skipped-model-upgrades.md`
- the fix-confirmed `/tmp/research_at_fix_confirmed.md`
