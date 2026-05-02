# Draft PR

- ship order: `01`
- depends on: `dojoengine/torii#428`
- branch: `refactor/centralize-rollback-cache-recovery`
- status: `ready to promote after dojoengine/torii#428 merges`
- upstream target: `dojoengine/torii:main`
- promotion target: cherry-pick `772f8216` from `wip/rob`
- scope: replace the manual rollback cache reset sequence with
  `Cache::reset_to_committed_storage()` without widening rollback semantics
- tasks: see `tasks/`
