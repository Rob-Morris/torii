# Torii cache-poison-on-rollback — production-readiness checklist

Tracking work to take the WIP patch (commits `84ab46a1`, `ff031a09`) to a mergeable PR.

Source of truth for the underlying bug:
`~/Development/Underware/pistols/torii-emulator/docs/shaping/research/torii-skipped-model-upgrades.md`.

## What the WIP already does

The two commits land the four-part patch identified in the research:

1. **`TaskManager::add_parallelized_event_with_dependencies` merges deps** into an existing task instead of just appending the event (`crates/processors/src/task_manager.rs`). Fixes the same-chunk historical-task ordering bug — earlier same-player events created the task before the upgrade was reached, so the later post-upgrade event ran against the old schema.
2. **`TaskNetwork` retains unresolved dependencies** via `pending_dependents` and activates them once the prerequisite is inserted (`crates/task-network/src/lib.rs`). Replaces the old "silently drop deps whose prerequisite doesn't exist yet" behavior. New unit tests for both the merge and the late-prerequisite paths.
3. **Engine rollback clears the model cache and balances diff** (`crates/indexer/engine/src/engine.rs`). Closes the model-cache poisoning window where a rolled-back chunk left the cache holding a post-upgrade schema while SQL had reverted.
4. **Processors read model definitions via `ctx.storage.model(...)` instead of `ctx.cache.model(...)`** (7 processor files). Required because (3) empties the cache after rollback; storage-backed lookup falls through to committed SQLite and repopulates the cache. Without this, retry hits `CacheError(ModelNotFound)`.

That is the runtime fix the research validated end-to-end with a Sepolia replay.

## What still needs doing for a production PR

Scope rule for the remaining work:
- Keep this patch narrowly focused on the two confirmed bug classes from the Pistols replay.
- Do not expand into unrelated torii cleanup or fetcher/provider issues just because they showed
  up during validation.
- The goal is to upstream the specific fix without causing regressions, not to make sweeping
  changes across torii.

### In-scope

- [x] **1. Restore typed model-not-found matching across the 7 processors.**
      The WIP changed `Err(CacheError::ModelNotFound(_)) if !namespaces.is_empty()` to a
      blanket `Err(_) if !namespaces.is_empty()`. With namespace filtering enabled, this now
      silently swallows *any* storage error (DB timeout, sqlx decode error, anything) as
      "model not found, skipping" and returns `Ok(())`. That is the same class of bug we are
      fixing — silently dropping events on transient failure.

      Approach (additive, non-breaking):
      - Add `ReadOnlyStorage::model_optional(world, selector) -> Result<Option<Model>, StorageError>`
        in `crates/storage/src/lib.rs`.
      - Implement on `Sql` via `fetch_optional` so "no rows" returns `Ok(None)` and any other
        sqlx error propagates. Mirror the cache fast path of the existing `model()` —
        cache hit returns `Ok(Some)`, cache miss falls through to DB.
      - Update the 7 processors:
        - `event_message.rs`, `store_del_record.rs`, `store_set_record.rs`,
          `store_update_member.rs`, `store_update_record.rs`, `upgrade_event.rs`,
          `upgrade_model.rs`
        - Pattern:
          ```rust
          let model = match ctx.storage.model_optional(world, selector).await? {
              Some(m) => m,
              None if !ctx.config.namespaces.is_empty() => {
                  debug!(...); return Ok(());
              }
              None => return Err(/* expected, propagate */),
          };
          ```
        - Pre-WIP behavior: `Err(CacheError::ModelNotFound)` with an empty namespace filter
          *propagated*; preserve that.

      Landed in the live worktree:
      - `ReadOnlyStorage::model_optional(...)`
      - `Sql::model_optional(...)`
      - typed `Error::ModelNotFound(Felt)` in processors
      - 7 processor call-site updates
      - sqlite tests covering `Ok(None)` and cache repopulation

- [x] **2. Token-registration cache rollback (research F1).**
      `mark_token_registered` mutates `ErcCache::token_id_registry` synchronously after
      enqueuing `register_token_contract` / `register_nft_token` (`crates/processors/src/erc.rs:432, 470`).
      Same shape as the model cache hazard: rollback drops the SQL but not the cache;
      retry sees `is_token_registered == true` and skips re-registration; the row never
      lands.

      Approach (additive):
      - Add `Cache::reset_token_registry(&self)` that repopulates `token_id_registry`
        from `storage.token_ids()` (matches the existing `ErcCache::new` pattern).
        A bare `clear()` would falsely report committed tokens as unregistered, because
        `is_token_registered` does *not* fall through to storage the way `Sql::model` does.
      - Call from the engine rollback arm in `crates/indexer/engine/src/engine.rs:236`
        alongside `clear_models` and `clear_balances_diff`.
      - The cache trait will need a handle to `Arc<dyn ReadOnlyStorage>` for the repopulate.
        `InMemoryCache` already holds one indirectly (only at `new`); plumb a stored
        `Arc<dyn ReadOnlyStorage>` field into `InMemoryCache` / `ErcCache` and reuse it.

      Landed in the live worktree:
      - `Cache::reset_token_registry(&self) -> Result<(), CacheError>`
      - `ErcCache` stores `Arc<dyn ReadOnlyStorage>` and rebuilds from `storage.token_ids()`
      - engine rollback now calls `reset_token_registry().await?` after clearing models/balances
      - cache unit test covering dropped uncommitted token marks after rollback

- [x] **3. Deterministic regression test for the rollback path.**
      The research calls this out explicitly. Without it, the bug regresses on any future
      engine rewrite. Shape:
      - tiny local Katana world with a model that can be additively upgraded
      - chunk containing `ModelUpgraded` followed by an event guaranteed to fail processing
      - assert first pass: `ALTER TABLE` enqueued, cache mutated, then `storage.rollback()` fires
      - assert post-rollback cache state is restored from committed storage (model cache empty
        or rebuilt; token registry repopulated)
      - assert second pass: `ALTER TABLE` is replayed and the column lands

      Place under `crates/indexer/engine/tests/` or co-located in
      `crates/processors/src/tests/` if a Katana fixture is easier there.

      Progress so far:
      - cache unit tests cover `clear_models()` and `reset_token_registry()` rollback semantics
      - sqlite tests cover `model_optional()` miss semantics and cache repopulation
      - a Katana-backed engine regression is now present in `crates/indexer/engine/src/test.rs`
        for the token-registry rollback path (`test_rollback_resets_token_registry_for_retry`)
      - that engine test has now passed locally end to end when run with:
        `PATH="$HOME/.asdf/shims:/opt/homebrew/bin:$PATH" KATANA_RUNNER_BIN=/Users/robmorris/Development/Underware/katana/target/debug/katana cargo test -p torii-indexer test_rollback_resets_token_registry_for_retry -- --nocapture`
      - a synthetic engine-level regression now covers the model-cache rollback path without any
        external Katana dependency:
        `PATH="$HOME/.asdf/shims:/opt/homebrew/bin:$PATH" cargo test -p torii-indexer test_rollback_replays_model_upgrade_after_cache_reset -- --nocapture`
      - both engine rollback regressions now pass together with:
        `PATH="$HOME/.asdf/shims:/opt/homebrew/bin:$PATH" KATANA_RUNNER_BIN=/Users/robmorris/Development/Underware/katana/target/debug/katana cargo test -p torii-indexer test_rollback_ -- --nocapture`

      The task-network unit tests added in the WIP cover items (1) and (2) of the runtime
      patch but not the full cache/rollback interaction in items (3) and (4).

- [ ] **4. Verification.**
      - `bash scripts/rust_fmt.sh --fix` clean
      - `PATH="/opt/homebrew/bin:$PATH" bash scripts/clippy.sh` clean
      - local Dojo fixture state had to be rebuilt for the workspace suite with:
        - `cd crates/types-test && PATH="$HOME/.asdf/shims:/opt/homebrew/bin:$PATH" sozo build -P dev`
        - `cd examples/spawn-and-move && PATH="$HOME/.asdf/shims:/opt/homebrew/bin:$PATH" sozo build -P dev`
      - targeted tests now passing:
        - `PATH="$HOME/.asdf/shims:/opt/homebrew/bin:$PATH" cargo test -p torii-cache -p torii-task-network -p torii-processors -- --nocapture`
        - `PATH="$HOME/.asdf/shims:/opt/homebrew/bin:$PATH" KATANA_RUNNER_BIN=/Users/robmorris/Development/Underware/katana/target/debug/katana cargo test -p torii-indexer test_rollback_resets_token_registry_for_retry -- --nocapture`
        - `PATH="$HOME/.asdf/shims:/opt/homebrew/bin:$PATH" cargo test -p torii-indexer test_rollback_replays_model_upgrade_after_cache_reset -- --nocapture`
        - `PATH="$HOME/.asdf/shims:/opt/homebrew/bin:$PATH" KATANA_RUNNER_BIN=/Users/robmorris/Development/Underware/katana/target/debug/katana cargo test -p torii-indexer test_rollback_ -- --nocapture`
        - `PATH="$HOME/.asdf/shims:/opt/homebrew/bin:$PATH" KATANA_RUNNER_BIN=/Users/robmorris/Development/Underware/katana/target/debug/katana cargo test -p torii-cache -p torii-task-network -p torii-sqlite model_optional -- --nocapture`
      - full workspace nextest was re-run with:
        `PATH="$HOME/.asdf/shims:/opt/homebrew/bin:$PATH" KATANA_RUNNER_BIN=/Users/robmorris/Development/Underware/katana/target/debug/katana cargo nextest run --all-features --workspace`
        and got past the earlier missing-world-state failures, but still stopped on 8
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
        `Provider(Other(TransportError(Json(Error("data did not match any variant of untagged enum JsonRpcResponse", line: 0, column: 0)))))`
      - those 8 `torii-indexer-fetcher` failures are confirmed pre-existing on clean
        `origin/main` under the current Katana/provider setup and are out of scope for this patch
      - replay validation completed against the local pre-critical Pistols Sepolia DB state:
        - source DB: `/private/tmp/pistols-torii-repro-patched.VuJ2O7/torii.db`
        - confirmed starting state:
          - world head `2262908`
          - `Config|0|0`
          - `PlayerActivityEvent|0|0`
        - fresh replay copy:
          `/private/tmp/pistols-torii-repro-recheck.l0J0eM`
        - config:
          `/Users/robmorris/Development/Underware/pistols/dojo/torii_sepolia_repro.toml`
        - run command:
          `PATH="$HOME/.asdf/shims:/opt/homebrew/bin:$PATH" RUST_LOG=info ./target/debug/torii --config /Users/robmorris/Development/Underware/pistols/dojo/torii_sepolia_repro.toml --db-dir /private/tmp/pistols-torii-repro-recheck.l0J0eM`
        - observed replay milestones:
          - crossed `2273149`
          - crossed `2283390`
          - crossed `2303872` before shutdown
        - post-run validation:
          - `pistols-Config` table includes `realms_address`
          - `pistols-PlayerActivityEvent` DDL includes `EnlistedRankedDuelist` in `activity_check`
          - `models` flags are now:
            - `Config|1|0`
            - `PlayerActivityEvent|0|1`
          - no `InvalidEnumSelector` surfaced in the captured replay logs
        - discrepancy resolved:
          - the earlier in-run `count(*) where activity='EnlistedRankedDuelist'` query was against
            the current-state table `[pistols-PlayerActivityEvent]`, not the append-only
            historical table
          - `set_event_message(...)` stores current event-message rows keyed by
            `poseidon_hash_many(keys)` and uses `ON CONFLICT(id) DO UPDATE`, so later
            `PlayerActivityEvent` values for the same key overwrite the current row
          - final current-state rows for the two affected players had advanced to
            `ChallengeCreated`, which is why the final current table showed `0` enlisted rows
          - the durable historical proof is in `event_messages_historical`, which now contains
            `7` persisted `PlayerActivityEvent` rows whose JSON payload includes
            `EnlistedRankedDuelist`, up to `2025-09-29T11:11:48+00:00`
          - this was only a validation/debugging interpretation issue in this run; it does not
            imply any additional torii patch change

- [ ] **5. PR hygiene.**
      - Squash WIP commits into logical units: (a) task-network late-dep + merge,
        (b) engine rollback + cache-clear + storage-backed model reads + new
        `model_optional`, (c) token-registry rollback, (d) regression test.
      - Cleanup check resolved:
        - keep the one-line `snapshot.version` arg ID change in `crates/cli/src/options.rs`
          because it avoids a clap arg-ID collision with the top-level built-in `version` flag
        - a new CLI test now locks that in:
          `args::test::test_clap_definition_is_valid`
        - `TaskNetwork::contains_key` in `crates/task-network/src/lib.rs` was unused and has been
          removed to keep the patch minimal
      - Do not broaden the PR to address the pre-existing `torii-indexer-fetcher` failures.
        Note them as existing validation noise in the current local Katana/provider environment,
        but leave `crates/indexer/fetcher` untouched for this fix series.
      - PR description: link the Pistols incident, summarize the four-part fix, call
        out behavior changes (silent skip now requires `Ok(None)` instead of any error).

### Definitions / call-site map

Files that change for item (1):
- `crates/storage/src/lib.rs` — `ReadOnlyStorage` trait + new `model_optional` default impl
  using existing `model` for back-compat? No — keep additive without default impl, force
  storage backends to opt in (only `Sql` exists in-tree).
- `crates/sqlite/sqlite/src/storage.rs` — `Sql::model_optional` mirroring `Sql::model`.
- 7 processor files listed above.

Files that change for item (2):
- `crates/cache/src/lib.rs` — new `Cache::reset_token_registry`, plus stored
  `Arc<dyn ReadOnlyStorage>` on `InMemoryCache` / `ErcCache`.
- `crates/indexer/engine/src/engine.rs` — call `cache.reset_token_registry().await` on
  rollback.

## Status (as of 2026-05-01)

WIP committed: `84ab46a1 wip`, `ff031a09 wip: simplify`, `9a0a91a3 wip: harden rollback cache recovery`.

Live worktree now adds the model-cache engine regression on top of the WIP commit snapshot.
Items 1, 2, and 3 are now covered. Item 4 is partially covered:
- format, clippy, targeted tests, and both rollback regressions are green
- full workspace nextest is still blocked by 8 `torii-indexer-fetcher` pending/preconfirmed
  tests failing on provider JSON-RPC response parsing against the current Katana setup
- replay validation from the research now passes on the decisive head/schema checks
- the only replay-note correction after that run was a query-surface mix-up
  (`[pistols-PlayerActivityEvent]` current state vs `event_messages_historical` append-only
  history), not a code issue

Item 5 (commit shaping / PR prep) is now the primary remaining work. The fetcher failures are
still worth noting in validation output, but they are pre-existing and should not pull this patch
into unrelated code.
