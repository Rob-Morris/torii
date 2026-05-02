# WIP

Current local task queue and follow-up notes.

Everything in this folder is local continuation material and should not be included in
upstream PR branches.

Contents:

- `00-first/`
  - highest-priority tasks to look at first
- `01-next/`
  - near-term follow-up tasks
- `02-later/`
  - lower-priority or parked follow-up tasks
- `90-rejected/`
  - rejected tasks or approaches that should not be reopened casually
- `prs/`
  - draft PR packets that bundle one or more assigned tasks

Conventions:

- one task per file
- each task keeps a stable `T###` identifier
- tasks move between priority buckets as priorities change
- tasks are the work map
- PR packets are the bundle
- unassigned tasks live in the priority buckets
- once a task is assigned to a draft PR, move it into that draft PR packet's `tasks/` folder

Task procedure:

1. Capture a new follow-up as a task file in the appropriate priority bucket.
2. Move the task between `00-first/`, `01-next/`, `02-later/`, and `90-rejected/` as priorities
   change.
3. When one or more tasks become real PR work, create a draft PR packet under
   `prs/NN-pr-<slug>/`.
4. Move the assigned task notes into `prs/NN-pr-<slug>/tasks/`.
5. Use the draft PR packet to track bundle-level notes, wording, and scope.
6. If a task is removed from that draft PR, move it back into the appropriate priority bucket or
   into `90-rejected/`.
7. When that PR is pushed upstream, move or recreate the finalized PR packet under
   `.dev.local/shipped/pr-<number>/`.

WIP PR packet structure:

- `prs/NN-pr-<slug>/README.md`
  - branch, status, and scope
- `prs/NN-pr-<slug>/pr-notes.md`
  - detailed notes for the bundle
- `prs/NN-pr-<slug>/pr-wording.md`
  - draft PR title/body wording
- `prs/NN-pr-<slug>/tasks/`
  - the assigned task notes included in that draft PR
