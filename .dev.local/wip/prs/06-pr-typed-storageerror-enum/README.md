# Draft PR

- ship order: `06`
- depends on: `none`
- branch: `refactor/typed-storageerror-enum`
- status: `queued behind 01 through 05 and ready to promote later`
- upstream target: `dojoengine/torii:main`
- promotion target: cherry-pick `506502a6` from `wip/rob`
- scope: replace boxed dynamic storage errors with a typed `StorageError` enum for model lookup and remove the additive `model_optional()` path
- tasks: see `tasks/`

Files:

- `pr-notes.md`
  - detailed draft PR notes: scope boundaries, exclusions, validation, open questions, and promotion notes
- `pr-wording.md`
  - draft PR title/body wording
- `tasks/`
  - assigned task notes included in this draft PR
