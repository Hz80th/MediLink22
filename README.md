# MediLink22: Engineering Management Simulation

Team repository for the MediLink appointment platform release cycle (SE Management assignment).
We are not building the full MediLink app. Each work item produces a small, reviewable artifact
(design note, mock API/JSON, SQL sketch, test plan, runbook) so that our workflow, reviews and
metrics are real.

## Team and roles

| Role | Member | Responsibilities |
|---|---|---|
| A: Engineering Manager / Team Lead | Tetant Tetant | Coordination, PR reviews, async/sync rules, coaching, report assembly. Owns 2 work items. Coaches rather than solving every technical problem. |
| B: Developer 1 | Mengqi Tong | Project board, workflow observations, incident RCA, slides. Owns 3 work items. |
| C: Developer 2 | Shenhuan Zhou | Definition of Done, risk register, dashboard, improvement experiment. Owns 3 work items. |

## Workflow

`Backlog → Ready → In Progress → Code Review → Verification → Done`

| Column | Entry rule |
|---|---|
| Backlog | Idea captured, not yet refined |
| Ready | Has owner, priority and testable acceptance criteria |
| In Progress | Owner is actively working on it (WIP limit 3) |
| Code Review | PR opened and linked to the issue |
| Verification | PR approved; owner checks every acceptance criterion and the DoD |
| Done | All acceptance criteria and DoD items ticked, PR merged |

## WIP limit: 3 cards in In Progress

We set the limit to 3, one card per person, because the last release showed developers starting new
tasks while their earlier work waited for review. With one card each, a developer who is blocked must
help unblock work (usually by reviewing) before pulling anything new, which keeps the review queue short.

## Definition of Done

See [docs/dod.md](docs/dod.md). A card cannot move to Done until every item is ticked.

## Daily updates

Asynchronous, posted as a comment on the pinned "Daily updates" issue. Every update cites issue numbers:

```
Completed: ...
Planned: ...
Blockers / help needed: ...
```

## Decisions

Team decisions are recorded in [docs/decisions.md](docs/decisions.md).
