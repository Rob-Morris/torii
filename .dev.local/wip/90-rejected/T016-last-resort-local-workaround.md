# T016 — Last-resort local workaround

Status:
- queue: `90-rejected`

Summary:
- catch the exact historical deserialization failure locally and drop that event instead of
  rolling back the whole chunk

Why rejected:
- emergency operator workaround, not an upstream-quality fix
- trades correctness in the affected historical stream for forward progress
- should only be considered as a temporary local escape hatch before the real fix is deployed
