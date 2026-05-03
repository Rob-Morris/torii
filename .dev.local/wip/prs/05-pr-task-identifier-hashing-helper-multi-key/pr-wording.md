# Title

`refactor: centralize multi-key task identifier hashing`

# Body

```md
## Motivation

- Four `store_*` processors still repeat the same inline `DefaultHasher` setup for `(from_address, keys[1], keys[2])`.
- T007 intentionally introduced only the single-key helper, so this repeated multi-key shape remains outside the shared helper path.

## What Changed

- Added `task_id_from_address_and_keys()` next to the existing task-ID helper in `task_manager`.
- Switched the four duplicated `store_*` task-identifier call sites to use that helper mechanically.
- Added order-sensitivity coverage for the multi-key helper.

## Scope

- Includes only the repeated `(from_address, keys[1], keys[2])` hashing pattern in the `store_*` processors.
- Excludes `IndexingMode::Latest(...)` hashing and other distinct task-ID shapes such as `event_message.rs`.

## For review

### Consequences

- The touched `store_*` task identifiers now share one helper instead of repeating inline hasher setup.
- No intended change to task-ID semantics.

### Risks

- No new known runtime risk beyond normal refactor risk.

### Validation

- `cargo test -p torii-processors task_id_from_address_and -- --nocapture`
- `cargo check -p torii-processors -p torii-indexer`
```
