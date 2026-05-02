# T006 — Tracing and logging hygiene

Status:
- queue: `02-pr-tracing-logging-hygiene`

Summary:
- clean up repeated selector hex formatting and reduce eager-format/cache-miss logging noise

Why this task exists:
- `format!("{:#x}", felt)` is repeated across many processor call sites
- the current `model_optional` cache-miss logging eagerly formats selectors and logs every miss
- after rollback the cache is intentionally cold, so this path is normal rather than alarming

Follow-up shape:
- introduce a small shared helper for selector formatting
- prefer lazy formatting in tracing fields where possible
- consider demoting normal cache-miss logging from `warn!` to `debug!`

Why next:
- real cleanup value
- low semantic risk
- easy follow-up PR candidate
