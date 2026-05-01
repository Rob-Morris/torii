# T009 — Typed `StorageError` enum

Status:
- queue: `02-later`

Summary:
- replace boxed dynamic storage errors with a typed enum across the storage surface

Why this task exists:
- the shipped fix needed additive `model_optional()` because current `StorageError` is not
  matchable
- a typed enum would make storage failure handling explicit and would likely remove the need for
  `model_optional()`

Why later:
- touches the whole storage call surface
- valuable, but much broader than the bugfix follow-up path
- deserves its own planning pass
