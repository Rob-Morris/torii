# T009 — Typed `StorageError` enum

Status:
- queue: `01-next`

Summary:
- replace boxed dynamic storage errors with a typed enum across the storage surface

Why this task exists:
- the shipped fix needed additive `model_optional()` because current `StorageError` is not
  matchable
- a typed enum would make storage failure handling explicit and would likely remove the need for
  `model_optional()`

Why next:
- touches the whole storage call surface
- valuable enough to queue ahead of the more speculative replay-hardening ideas
- broad enough that it still deserves its own planning pass before implementation
