# T011 — Version historical event schemas by resource contract

Status:
- queue: `02-later`

Summary:
- key historical event schemas by resource contract identity, not just `(world, selector)`

Why this task exists:
- event resources can move across different resource contracts over time
- selector-only schema identity may be too coarse for certain historical replay cases

Why later:
- the captured bug was not caused by this
- this is structural hardening for a different, currently theoretical failure class
