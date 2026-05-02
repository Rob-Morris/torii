# Title

`refactor: share ReadOnlyStorage test stub`

# Body

```md
## Motivation

- Cache and sqlite tests each carry a near-identical full `ReadOnlyStorage` stub.
- Sharing one stub keeps storage-adjacent tests aligned as the trait evolves without widening this into a broader test-helper framework.

## What Changed

- Added a shared minimal `ReadOnlyStorage` test stub to `torii-storage`.
- Exposed it to dependent tests through a `test-utils` feature.
- Switched the cache and sqlite tests that used local stubs to the shared helper.

## Scope

- Includes only the shared storage test stub and the cache/sqlite test migrations to it.
- Excludes broader storage test utilities and production storage behavior changes.

## For review

### Consequences

- Cache and sqlite tests now share one stub implementation for the minimal `ReadOnlyStorage` surface they exercise.
- Future trait-shape changes should only need one stub update instead of duplicated edits in both crates.
- No intended production behavior change.

### Risks

- No new known production/runtime risk; this is a test-support refactor.

### Validation

- `cargo test -p torii-storage`
- `cargo test -p torii-cache -- --nocapture`
- `cargo test -p torii-sqlite model_optional -- --nocapture`
- `PATH="/opt/homebrew/bin:$PATH" cargo +nightly-2025-05-01 clippy -p torii-storage -p torii-cache -p torii-sqlite --tests -- -D warnings -D future-incompatible -D nonstandard-style -D rust-2018-idioms -D unused -D missing-debug-implementations -A clippy::uninlined_format_args`
- `git diff --check`
```
