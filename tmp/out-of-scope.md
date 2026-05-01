# Out-of-scope hardening — second-pass candidates

Items identified during the cache-poison-on-rollback investigation that are real hardening
wins but should not block the production PR. Each entry includes a short rationale so a
follow-up author can pick them up.

References: research at `~/Development/Underware/pistols/torii-emulator/docs/shaping/research/torii-skipped-model-upgrades.md`,
section "Appendix C — Alternative and rejected fix approaches".

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

## 6. F7 — generic "ignore unknown enum selector" parser patch

Explicitly **not recommended** in the research. Once enum decoding accepts an unknown
selector, torii no longer knows how many felts to consume for the variant payload. For
unit variants this might appear harmless; for payload-bearing variants it desynchronizes
the rest of the decode stream and creates harder-to-debug corruption.

Listed only so a future contributor doesn't re-propose it.

## 7. Tracing-field hex-format helper

The `format!("{:#x}", felt)` pattern appears across roughly 30 call sites in
`crates/processors/`, including the diagnostic logs added in the WIP. A small helper
(`fn hex(f: Felt) -> String` or `fn hex_args(...) -> impl Display`) would deduplicate the
boilerplate and avoid the eager allocation cost in tracing fields.

Why later: pure cleanup, orthogonal to the bug. The current code already follows the
established convention in `crates/processors/src/erc.rs`.

## 8. Combined `Cache::clear_all()` for rollback

Engine rollback currently calls three separate cache-reset methods. A single
`Cache::reset_to_committed_storage()` would centralize the rollback-aware semantics so a
future cache field is harder to forget about.

Why later: ergonomic refactor; semantically equivalent to today's call list once item (2)
in `production-readiness.md` lands. Worth doing, but trivially fix-forwardable later.
