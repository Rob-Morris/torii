# Out-of-scope follow-up priority guide

This note explains how to prioritize the candidate follow-up items listed in
`tmp/out-of-scope.md`.

Use it for sequencing. Keep `tmp/out-of-scope.md` itself as the shorter inventory of
candidate work.

## How to use the list

This is not just "everything we could do." It is ordered by pragmatic follow-up value
after the upstream bugfix PR:

- prefer items that directly reduce the chance of another rollback/cache regression
- prefer small cleanups only when they are genuinely cheap and low-risk
- keep larger semantic or architectural changes separate from the bugfix series
- do not treat every plausible hardening idea as near-term roadmap

Two different orderings matter:

- engineering-priority order: what is most worth doing for torii after this PR lands
- contributor-friendly order: what is easiest to turn into small safe cleanup PRs

## Priority guide

### Highest-value follow-up

1. `#7 Centralize rollback cache recovery API`

Why:
- most directly tied to the bug class this work just fixed
- reduces the chance that a future cache-backed field is added without rollback recovery
- keeps engine rollback code and regression tests from drifting apart

Why not in this PR:
- it is API / maintainability cleanup, not part of the correctness fix
- the current explicit rollback sequence is already short and correct

### Small worthwhile cleanup PRs

2. `#6 Tracing / logging hygiene around selector formatting and cache misses`
3. `#8 Shared task_identifier hashing helper`
4. `#9 Shared ReadOnlyStorage test stub`

Why this bucket:
- each is real cleanup value without changing runtime semantics
- each can be reviewed independently
- they are lower impact than `#7`, but much smaller than `#1`

Ordering within the bucket:
- `#6` is the smallest worthwhile cleanup
- `#8` is a broader mechanical consistency refactor
- `#9` is useful, but adds shared test-support surface and is slightly less urgent

### High-payoff but larger planned refactor

5. `#1 Make StorageError a typed enum`

Why:
- highest long-term design payoff on the list
- would remove the need for `model_optional`
- makes storage failures matchable and explicit across the workspace

Why later:
- touches the whole storage call surface
- deserves its own planning pass, not a drive-by cleanup

### Keep noted, but do not actively schedule

6. `#2 F2 — defer cache writes until after storage.execute()`
7. `#3 F4 — version historical event schemas by resource contract`
8. `#4 F5 — self-heal on enum mismatch`
9. `#5 F6 — strict block-aligned schema reads by default`

Why this bucket:
- these are plausible hardening directions, but not currently justified enough to treat as
  near-term follow-up work
- `#2` and `#5` are larger semantic changes that need focused replay validation
- `#3` addresses a theoretical failure class we have not yet seen in the wild
- `#4` is defense-in-depth, not a root fix

## Contributor-friendly micro-PR order

If the goal is community-friendly micro-PRs rather than engineering priority, the easiest
stack is:

1. `#6 Tracing / logging hygiene around selector formatting and cache misses`
2. `#9 Shared ReadOnlyStorage test stub`
3. `#8 Shared task_identifier hashing helper`
4. `#7 Centralize rollback cache recovery API`
