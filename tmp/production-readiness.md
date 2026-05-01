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

- [ ] **3. Deterministic regression test for the rollback path.**
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
      - still missing as fully verified coverage: a matching engine/chunk-level replay regression
        for the model-cache rollback path

      The task-network unit tests added in the WIP cover items (1) and (2) of the runtime
      patch but not the full cache/rollback interaction in items (3) and (4).

- [ ] **4. Verification.**
      - `bash scripts/rust_fmt.sh --fix` clean
      - `PATH="/opt/homebrew/bin:$PATH" bash scripts/clippy.sh` clean
      - targeted tests now passing:
        - `PATH="$HOME/.asdf/shims:/opt/homebrew/bin:$PATH" cargo test -p torii-cache -p torii-task-network -p torii-processors -- --nocapture`
        - `PATH="$HOME/.asdf/shims:/opt/homebrew/bin:$PATH" KATANA_RUNNER_BIN=/Users/robmorris/Development/Underware/katana/target/debug/katana cargo test -p torii-indexer test_rollback_resets_token_registry_for_retry -- --nocapture`
        - `PATH="$HOME/.asdf/shims:/opt/homebrew/bin:$PATH" KATANA_RUNNER_BIN=/Users/robmorris/Development/Underware/katana/target/debug/katana cargo test -p torii-cache -p torii-task-network -p torii-sqlite model_optional -- --nocapture`
      - `KATANA_RUNNER_BIN=katana cargo nextest run --all-features --workspace` green
      - Re-run the patched Sepolia replay from pre-critical head `2262908` per the
        research's "Patched replay" validation; cross the trigger window cleanly.

- [ ] **5. PR hygiene.**
      - Squash WIP commits into logical units: (a) task-network late-dep + merge,
        (b) engine rollback + cache-clear + storage-backed model reads + new
        `model_optional`, (c) token-registry rollback, (d) regression test.
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

WIP committed: `84ab46a1 wip`, `ff031a09 wip: simplify`.

Live worktree now covers items 1 and 2, plus targeted unit tests for the new cache/storage
paths and a runtime-verified engine-level rollback regression for the token-registry path.
Remaining blockers are item 3 (completion of model-cache engine-level regression coverage),
item 4 (full validation suite + replay), and item 5 (commit shaping / PR prep).
