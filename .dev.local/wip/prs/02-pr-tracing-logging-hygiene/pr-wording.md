# Title

`refactor: tighten tracing and cache-fallback logging`

# Body

```md
## Motivation

- Processor-side logging in the model/event paths currently open-codes felt hex formatting in several places.
- Expected cache fallback after rollback currently logs too loudly and eagerly formats selector details even when the fallback is normal.

## What Changed

- Added a shared lazy `hex_felt()` display helper in `torii_storage::utils`.
- Switched selected processor-side logging call sites in `metadata_update`, `event_message`, `register_external_contract`, and `upgrade_event` to use that helper instead of inline `format!("{:#x}", ...)`.
- Demoted `model_optional()` and `models()` cache-fallback logs in `torii-sqlite` from `warn!` to `debug!`.
- Replaced eager selector-list formatting in the `models()` cache-fallback path with count-based fields.

## Scope

- Includes only logging/tracing hygiene in the touched processor event/model paths and sqlite cache-fallback logs.
- Excludes fetcher logging cleanup, repo-wide felt-formatting cleanup, and any change to cache or model-resolution semantics.

## For review

### Consequences

- Expected cache fallback after rollback is quieter in normal operation.
- The touched processor-side logs still emit felt identifiers in hex form, but now use one shared formatter instead of repeated inline formatting.

### Risks

- No new known runtime risk beyond log content and level.

### Validation

- `cargo check -p torii-processors -p torii-sqlite -p torii-storage`
- `cargo test -p torii-sqlite model_optional -- --nocapture`
- `PATH="/opt/homebrew/bin:$PATH" cargo test -p torii-storage`
```
