# Out-of-scope hardening — second-pass candidates

Items identified during the cache-poison-on-rollback investigation that are real hardening
wins but should not block the production PR. Each entry includes a short rationale so a
follow-up author can pick them up.

References:
- research at `~/Development/Underware/pistols/torii-emulator/docs/shaping/research/torii-skipped-model-upgrades.md`
- rejected / emergency-only ideas moved to `tmp/rejected-approaches.md`
- prioritization and sequencing guidance moved to `tmp/out-of-scope-priority-guide.md`

## 1. Make `StorageError` a typed enum (workspace refactor)

Today `StorageError = Box<dyn Error + Send + Sync>` (`crates/storage/src/lib.rs:28`). That
is why the production-readiness PR has to add a sibling `model_optional()` to expose
"not found" as `Ok(None)` — there is no way to match `StorageError::ModelNotFound`.

A typed enum (variants for `NotFound`, `Sqlx`, `Provider`, `Decode`, …) would let every
storage method return a single `Result<T, StorageError>` and let callers match on the
variant. It would also obviate `model_optional`. The cost is touching every storage call
site in the workspace.

Why later: the additive `model_optional` fixes the immediate bug class without changing
existing return types. The enum refactor is a bigger surgery whose value compounds across
the whole storage surface, not just `model()`.

## 2. F2 — defer cache writes until after `storage.execute()`

The cleanest version of "cache state matches committed SQL state" is to never mutate the
cache before the SQL transaction commits. Today processors call
`ctx.cache.register_model(...)` immediately after enqueuing
`storage.register_model(...)`; the cache is mutated in-process, the SQL is a queued
message that only commits when the engine calls `self.storage.execute().await` at the end
of the chunk.

A staged cache (per-chunk staging buffer, flushed on commit, dropped on rollback) makes the
hazard impossible by construction. It also closes the smaller window where two parallel
tasks in the same chunk can read a "future" schema from the cache before SQL commit.

Why later: the runtime fix in this PR (clear cache on rollback, read via storage that falls
through to committed DB) is sufficient for the observed bug. F2 is a bigger restructure
that changes a hot-path read pattern (`set_entity` would no longer see same-chunk schema
upgrades from the cache) and needs careful thought about how parallel processors in the
same chunk see each other's writes.

## 3. F4 — version historical event schemas by resource contract

`ModelCache` is keyed by `(world_address, selector)` only. Event resources (e.g.
`pistols-PlayerActivityEvent`) move across distinct resource contract addresses over time
on chain, and the old contracts stay deployed with their old schemas. Torii squashes them
into a single per-`(world, selector)` row.

The hardening direction is to key event schemas by
`(world_address, selector, contract_address)` (or `class_hash`) so historical replay can
deserialize against the resource version that actually emitted the payload, and
non-additive event schema changes become safe.

Why later: the bug captured in the research replay is *not* explained by selector-only
versioning — the failing event was decoded against the right schema slot, just stale-cached.
F4 is a structural improvement that prevents a different (currently theoretical) failure
class.

## 4. F5 — self-heal on enum mismatch

If `EventMessageProcessor` catches `PrimitiveError::InvalidEnumSelector`, it could refetch
the event schema from chain for that selector, update cache and storage if the schema
diverged, and retry the deserialize once.

Why later: this is a defense-in-depth backstop, not a root fix. The four-part patch in this
PR removes the situation that produces the mismatch in the first place. F5 is most useful
as protection against future similar bugs.

## 5. F6 — strict block-aligned schema reads by default

`strict_model_reader` defaults to `false` (`crates/processors/src/lib.rs:60`,
`crates/cli/src/options.rs:294`), so `register_model` / `upgrade_model` /
`register_event` / `upgrade_event` fetch schema at the provider's *latest* block instead
of `BlockId::Number(ctx.block_number)`. For historical replay this can pull a future schema
snapshot.

Worth testing whether flipping this to `true` (or making it configurable per-network)
removes another source of schema/state divergence.

Why later: lower-confidence; the captured failure was about cache poisoning, not about the
reader pulling a wrong snapshot. Worth a focused experiment, not blocking the rollback fix.

## 6. Tracing / logging hygiene around selector formatting and cache misses

The `format!("{:#x}", felt)` pattern appears across roughly 30 call sites in
`crates/processors/`, including the diagnostic logs added in the WIP. A small helper
(`fn hex(f: Felt) -> String` or `fn hex_args(...) -> impl Display`) would deduplicate the
boilerplate and avoid the eager allocation cost in tracing fields.

This also pairs naturally with the pre-existing `model_optional` cache-miss `warn!` in
`crates/sqlite/sqlite/src/storage.rs:72-77`, which eagerly formats the selector and logs
every cache miss at warning level. After rollback the cache is intentionally cold, so a
follow-up cleanup could:

- switch the selector field to `%selector` for lazy formatting
- demote the log to `debug!` if "cache miss after rollback" is considered normal
- optionally reuse the same helper for the remaining eager hex-format call sites

Why later: pure observability / cleanup work, not a correctness fix. The current code
already follows the established convention in `crates/processors/src/erc.rs`.

## 7. Centralize rollback cache recovery API

Engine rollback currently calls three separate cache-reset methods in order:
`clear_balances_diff` + `clear_models` + `reset_token_registry`. A single
`Cache::reset_to_committed_storage()` would centralize the rollback-aware semantics so a
future cache field is harder to forget about.

This would also let the engine regression tests call the same recovery path instead of
recreating the sequence inline, which reduces the chance of test drift if rollback
recovery grows a fourth step later.

As part of the same refactor, `ErcCache.storage: Arc<dyn ReadOnlyStorage>` could be
reconsidered. Today that field exists purely so `reset_token_registry(&self)` can call
`storage.token_ids()` later. If the reset path were centralized, the trait shape could
instead pass read-only storage into the reset call directly and keep the cache struct a
little narrower.

Why later: maintainability improvement, not a correctness fix. The current explicit
rollback sequence is short, readable, and already correct for this PR.

## 8. Shared `task_identifier` hashing helper

Every event processor reimplements the same `DefaultHasher` over `(from_address, key)`
pattern in its `task_identifier` / `task_dependencies` impls
(`crates/processors/src/processors/{store_set_record,store_del_record,store_update_record,store_update_member,upgrade_event,upgrade_model,event_message}.rs`).
The new rollback regression test in `crates/indexer/engine/src/test.rs` had to recreate
the same shape as `hashed_task_identifier`. Extract a single
`pub fn hash_task_id(parts: &[Felt]) -> TaskId` next to `TaskId` in
`crates/processors/src/task_manager.rs` and rewrite all call sites.

Why later: touches every event-processor file as a separate refactor; out of scope for the
rollback PR but a natural follow-up.

## 9. Shared `ReadOnlyStorage` test stub

The new rollback work introduced two near-identical full-trait stubs:
`StubStorage` in `crates/cache/src/lib.rs` tests and `EmptyStorage` in
`crates/sqlite/sqlite/src/storage.rs` tests. Both hand-roll all 17 `ReadOnlyStorage`
methods with `unimplemented!()` for the long tail. Promote a shared
`pub mod testing { pub struct StubReadOnlyStorage { ... } }` (gated on
`#[cfg(any(test, feature = "testing"))]`) in `crates/storage` so future test crates do
not need to re-stub the whole trait every time it grows.

Why later: feature-gated test surface change touching `torii_storage`; orthogonal to
rollback recovery.
