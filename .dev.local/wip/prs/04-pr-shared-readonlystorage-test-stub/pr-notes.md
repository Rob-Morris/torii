# Draft PR notes

- TODO: remove sections that do not apply, add sections if this PR needs them, and prefer omission over filler.

## Branch

- intended upstream branch: `refactor/shared-readonlystorage-test-stub`
- current integration branch: `wip/rob`

## Status

- draft packet created
- code changes ready on `wip/rob`
- targeted validation passed locally

## Ordering

- ship order: `04`
- no known upstream dependency on `01`, `02`, or `03`

## Promotion

- wait for the code commit on `wip/rob`
- branch from fresh `origin/main`
- cherry-pick the recorded T008 code commit
- push `refactor/shared-readonlystorage-test-stub`
- open the upstream PR against `dojoengine/torii:main`

## Context

- cache and sqlite test modules each carry a near-identical full `ReadOnlyStorage` stub
- recent rollback/cache work made the duplication more visible, and every future trait change would force both stubs to grow in lockstep

## Motivation

- storage-adjacent tests should share one minimal `ReadOnlyStorage` stub instead of restubbing the full trait per crate
- the helper should stay local to test builds in dependent crates rather than widen into a general runtime utility

## What Changed

- add a shared `ReadOnlyStorage` test stub in `torii-storage`
- make it available to dependent tests via a `test-utils` feature
- switch cache and sqlite tests to the shared helper

## Consequences

- cache and sqlite tests will share one stub implementation for `models`, `model_optional`, and `token_ids`
- future `ReadOnlyStorage` trait changes should only need one stub update instead of duplicated edits in each crate
- no intended production behavior change

## Risks

- low-risk test-support refactor

## Scope

- shared `ReadOnlyStorage` test helper in `torii-storage`
- cache/sqlite test migration to that helper

## Exclusions

- no broader storage test-support framework
- no production `ReadOnlyStorage` behavior changes
- no processor/indexer test cleanup outside the duplicated storage stubs

## Validation

- `cargo test -p torii-storage`
- `cargo test -p torii-cache -- --nocapture`
- `cargo test -p torii-sqlite model_optional -- --nocapture`
- `PATH="/opt/homebrew/bin:$PATH" cargo +nightly-2025-05-01 clippy -p torii-storage -p torii-cache -p torii-sqlite --tests -- -D warnings -D future-incompatible -D nonstandard-style -D rust-2018-idioms -D unused -D missing-debug-implementations -A clippy::uninlined_format_args`
- `git diff --check`

## Open questions

- none at the moment

## Notes

- prefer the narrowest helper surface that removes the duplicated stubs
