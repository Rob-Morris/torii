# T004 — Validation and replay confirmation

Status:
- shipped in PR `#428`

Primary outcome:
- the final shaped branch was validated locally with targeted tests, lint/format checks, and a
  replay through the known bad Sepolia window

What this captured:
- passing targeted validation for the task-network and rollback-fix surface
- confirmation that the remaining `torii-indexer-fetcher` failures were pre-existing on clean
  `origin/main`
- replay confirmation that the patched branch crossed the bad window, landed the expected schema
  changes, and did not reproduce `InvalidEnumSelector`

Why it mattered:
- this is the proof packet for the shipped PR, not just incidental local verification

See also:
- `.dev.local/shipped/pr-428/pr-notes.md`
