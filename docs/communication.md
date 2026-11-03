# Async vs Sync Communication

Our default is asynchronous: written, on GitHub, so it leaves a record and nobody has to be online at the same time.

## Stays async

| What | Where |
|---|---|
| Daily updates (Completed / Planned / Blockers, citing issue numbers) | Comment on pinned issue #9 |
| Code review feedback | PR review comments, checked against docs/dod.md |
| Questions about a work item | Comment on that issue |
| High and Normal production-support requests | GitHub issue, per docs/support-runbook.md |
| Decision records | docs/decisions.md |

## Switch to sync: two triggers

**Trigger 1: A PR review goes back and forth more than 3 rounds without agreement.**
Action: 15-minute call between author and reviewer, then the outcome is written as a PR comment.
Why sync: after 3 written rounds the disagreement is usually about assumptions, not the code. Each async round costs hours of waiting, and a short conversation finds the misunderstanding faster than a fourth comment.

**Trigger 2: A Critical production incident (see docs/support-runbook.md).**
Action: immediate call with the support person of the day and the EM. The incident issue is updated as the written record.
Why sync: patients are affected right now. Waiting for written replies delays the decision about who fixes it and whether new work pauses, and that decision needs to be made in minutes.

## Rule for every sync call

The outcome is written down afterwards (PR comment, issue comment, or docs/decisions.md). If it is not written down, it was not decided.
