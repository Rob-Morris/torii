# .dev.local

Local-only notes, research, and PR records for ongoing work around this repository.

This directory is not upstream PR material.

Subfolders:

- `wip/`
  - current local task queue and follow-up notes
- `research/`
  - research documents with date-prefixed filenames
- `shipped/`
  - shipped upstream PR packets

Conventions:

- `wip/` contains:
  - `README.md`
  - priority buckets for task files
  - `prs/`
    - draft PR packets that bundle one or more assigned task notes
    - `pr-template/`
      - reusable template for a new draft PR packet
- each task note keeps a stable `T###` identifier and can move between priority buckets
- `shipped/` contains one folder per shipped PR:
  - `pr-<number>/`
- each shipped PR folder should contain:
  - `README.md`
  - `pr-notes.md`
  - `pr-wording.md`
- a PR folder may also contain:
  - `tasks/`
    - completed task notes that shipped as part of that PR
