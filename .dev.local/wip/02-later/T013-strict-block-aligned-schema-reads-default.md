# T013 — Strict block-aligned schema reads by default

Status:
- queue: `02-later`

Summary:
- evaluate making strict block-aligned schema reads the default for historical replay

Why this task exists:
- current non-strict behavior can fetch a future schema snapshot during historical replay
- that may be another source of schema/state divergence worth testing

Why later:
- lower-confidence hypothesis
- the captured bug was about rollback/cache poisoning, not this reader mode
- worth a focused experiment, not a follow-up that blocks other work
