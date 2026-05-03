# Draft PR

- ship order: `05`
- depends on: `03-pr-task-identifier-hashing-helper`
- branch: `refactor/task-identifier-hashing-helper-multi-key`
- status: `queued behind 01 through 04 and ready to promote later`
- upstream target: `dojoengine/torii:main`
- promotion target: cherry-pick `73393d1f` from `wip/rob`
- scope: centralize the repeated `(from_address, keys[1], keys[2])` task-ID hashing pattern in the `store_*` processors without widening helper semantics
- tasks: see `tasks/`

Files:

- `pr-notes.md`
  - detailed draft PR notes: scope boundaries, exclusions, validation, open questions, and promotion notes
- `pr-wording.md`
  - draft PR title/body wording
- `tasks/`
  - assigned task notes included in this draft PR
