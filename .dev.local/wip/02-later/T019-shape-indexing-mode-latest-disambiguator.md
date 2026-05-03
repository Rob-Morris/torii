# T019 — Shape `IndexingMode::Latest` disambiguator hashing

Status:
- queue: `02-later`
- prerequisite: `05-pr-task-identifier-hashing-helper-multi-key` ships upstream
  first so the framing of this work is clearly "next step", not "scope creep
  on 05"

Summary:
- decide the right home for the `event.keys[0] -> u64` hash that the four
  `Store*Record` processors hand-roll inside `indexing_mode(...)` for
  `IndexingMode::Latest(...)`

## Observation

- four `Store*Record::indexing_mode` impls share the same shape:
  `let mut hasher = DefaultHasher::new(); event.keys[0].hash(&mut hasher); IndexingMode::Latest(hasher.finish())`
- this is structurally identical to the duplication T007/T017 cleaned up for
  task identifiers, but it is a different domain (no `from_address`, single
  key, used for latest-mode disambiguation rather than scheduling)
- explicitly excluded from PR `05`'s scope ("not a generalized hashing
  redesign") — to be revisited as its own packet, not folded in

## Interventions to choose between

1. extract a thin helper, e.g. `task_manager::latest_disambiguator(key: Felt) -> u64`
   - pros: minimal blast radius, mechanical change, mirrors T007/T017
   - cons: very thin wrapper around 4 lines; weak abstraction; if any callsite
     ever needs a different key index or shape, the parameter sneaks back in
     and the helper degenerates

2. push the hashing into `IndexingMode::Latest` itself (e.g.
   `IndexingMode::Latest::from_key(Felt)` or change the variant payload to
   `Felt`)
   - pros: removes the lossy `Felt -> u64` step from every caller; better
     semantic alignment (callers say "this is the latest entry for this key",
     not "here's a hash")
   - cons: bigger change touching the type itself and every consumer of
     `IndexingMode::Latest`; needs an audit of every site that constructs or
     pattern-matches the variant

3. do nothing
   - pros: zero churn for what is currently inert duplication
   - cons: leaves the only remaining hand-rolled hashing pattern in the
     processors module after T007/T017 cleanup

## Tradeoffs to weigh before doing the work

- task IDs are in-process only and never persisted, so cross-version
  `DefaultHasher` instability does not bite — that is *not* a reason to keep
  hashing centralized
- the four sites are textually identical today; confirm they are also
  semantically identical (always `keys[0]`, always the same disambiguator
  intent) before assuming a single helper fits
- option 2 is the design-correct intervention; option 1 is the cheap
  intervention. picking the cheap one risks paving over a real shaping
  question with a thin wrapper that future work will have to revisit
- upstream review economics: a tiny refactor PR with no functional driver may
  read as drive-by; bundling with a real reason to touch `IndexingMode`
  (option 2 plus a motivating change) lands more cleanly

## Why later

- no functional driver; pure cleanup
- option 2 is a type-shape change that needs its own design pass and audit,
  not a same-day mechanical edit
- deliberately deferred by PR `05`'s exclusions to keep that packet narrow

## Resolve before implementing

- option 1 vs option 2 vs do-nothing
- if option 2: does `IndexingMode::Latest` carry `Felt` directly, or expose a
  constructor that hashes? (the former is cleaner if no consumer needs the
  pre-hashed `u64`)
- if option 1: confirm helper signature and that no existing call site needs
  a different key index
