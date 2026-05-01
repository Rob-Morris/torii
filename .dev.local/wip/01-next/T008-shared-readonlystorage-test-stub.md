# T008 — Shared `ReadOnlyStorage` test stub

Status:
- queue: `01-next`

Summary:
- add a shared test stub for `ReadOnlyStorage` instead of restubbing the full trait in each crate

Why this task exists:
- the rollback work introduced near-identical full-trait stubs in multiple test modules
- every future `ReadOnlyStorage` trait change forces those local stubs to grow too

Likely shape:
- add a gated testing stub in `crates/storage`
- reuse it from cache/sqlite tests and future storage-adjacent test code

Why next:
- useful test-surface cleanup
- moderate scope, but still a contained follow-up
