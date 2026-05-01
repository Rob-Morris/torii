# T015 — Generic ignore-unknown-enum-selector parser patch

Status:
- queue: `90-rejected`

Summary:
- swallow unknown enum discriminants generically so replay can keep moving

Why rejected:
- once torii accepts an unknown enum selector, it no longer knows how many felts to consume for
  that variant payload
- for payload-bearing variants this can desynchronize the decode stream and create silent
  corruption that is harder to debug than a hard failure
- if any workaround is ever needed, it must be narrow and model-specific, not a generic parser
  behavior change
