# T017 — Task identifier hashing helper for multi-key patterns

Status:
- queue: `00-first`

Summary:
- extend the task-identifier hashing helper to cover the `(from_address, keys[1], keys[2])`
  three-input pattern still duplicated across the store_* event processors

Why this task exists:
- T007 (shipped via PR packet `03-pr-task-identifier-hashing-helper`) intentionally scoped its
  helper to the two-input `(from_address, key)` case to keep task semantics unchanged
- the three-input pattern remains duplicated in:
  - `crates/processors/src/processors/store_set_record.rs` (`task_identifier`)
  - `crates/processors/src/processors/store_del_record.rs` (`task_identifier`)
  - `crates/processors/src/processors/store_update_member.rs` (`task_identifier`)
  - `crates/processors/src/processors/store_update_record.rs` (`task_identifier`)
- `event_message.rs` uses a related but distinct three-input shape
  (`from_address`, `keys[1]`, `poseidon_hash_many(deserialized_keys)`); evaluate whether it
  fits the same helper or warrants its own
- every future change to the hashing shape must be repeated in each call site

Likely shape:
- add a sibling helper in `crates/processors/src/task_manager.rs`, e.g.
  `task_id_from_address_and_keys(from_address: Felt, keys: &[Felt]) -> TaskId`,
  preserving the existing `DefaultHasher` ordering semantics
- rewrite the four `task_identifier` call sites mechanically
- decide separately whether `event_message.rs` adopts the helper or keeps its bespoke shape
- add an order-sensitivity test mirroring the T007 test

Out of scope:
- the `IndexingMode::Latest(...)` hashing in the same files — different semantics
  (per-event uniqueness, not task identity); leave alone unless a separate task is opened

Why next:
- direct continuation of T007 with the same low-risk, mechanical character
- removes the last instances of the inline `DefaultHasher` task-identifier pattern
- unblocks dropping the `use std::hash::{DefaultHasher, Hash, Hasher};` imports from the
  store_* processor files once the `IndexingMode::Latest` hashing question is also resolved

Dependencies:
- T007 must ship upstream first so the helper module is in `dojoengine/torii:main`
