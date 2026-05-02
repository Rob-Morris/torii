# T018 — Automate PR promotion from draft packet

Status:
- queue: `02-later`

Summary:
- add a local helper that turns a draft PR packet plus recorded promotion target into a clean PR branch

Why this task exists:
- the current promotion flow is manual:
  - branch from the intended base
  - cherry-pick the recorded code commit
  - keep local-only packet files out of the promoted branch
  - push/open the upstream PR
- that process is workable, but it is easy to make mistakes with commit hashes, branch names, or local-only file inclusion once several queued PR packets exist

Likely shape:
- read `README.md` / `pr-notes.md` from a draft PR packet
- create a fresh branch from the recorded upstream target or base
- cherry-pick the recorded promotion commit
- optionally emit the PR body from `pr-wording.md`
- refuse to proceed if the packet metadata is incomplete or points at notes-only commits

Why later:
- valuable local tooling, but it does not unblock the code follow-up queue
- current manual promotion is still acceptable while the queue is small
