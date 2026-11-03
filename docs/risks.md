# Risk Register: MediLink Release

**Scope:** Risks identified from the scenario and the team's simulated workflow on Nov 3, 2026.

| ID | Risk | Likelihood | Impact | Mitigation | Evidence | Status |
|---|---|---|---|---|---|---|
| R1 | SMS/email provider SDK is retired; reminders stop sending | High | High | Issue #5 upgrade plan (`docs/maintenance/messaging-sdk-upgrade.md`): document current version + retirement date, list breaking changes, list all SDK call sites, write rollback plan | Issue #5 (P2, maintenance) | Open |
| R2 | Two patients book the same slot concurrently | Medium | High | Partial unique index (authoritative) + atomic check-and-insert (friendly path); unique-violation handled as "This slot was just taken" | Issue #3 (P1 defect); PR #15 merged; `docs/defects/double-booking.md` | Mitigated |
| R3 | PRs sit open because nobody notices they are ready for review | Medium | High | Rule (13:35): opener posts PR number on issue #9 and @mentions the reviewer; reviewer gives a first review within 30 min during work blocks; measured via Part H experiment | PR #15 first-review wait (see review log); #9 follow-up comment | In progress |
| R4 | Team members cannot start work because they are not repo collaborators | High | High | Tocky25 invited and now a Collaborator; assignable issues unblocked | Day 1 blocker flagged on #9 ("Mengqi not a collaborator"); PR #11 / #17 show Collaborator badge | Resolved |
| R5 | Critical incidents are handled ad hoc without defined ownership | Medium | High | Support runbook (issue #7, PR #12 merged): severity table (Critical 15 min / High 2 h), on-call rotation, 1 h escalation to backup + EM; Critical incidents switch to sync per `docs/communication.md` | Issue #7; `docs/support-runbook.md`; `docs/communication.md`; incident #18 acknowledged within 15 min on Nov 3 | In progress |
| R6 | Compressed 3-session cycle misses the hard deadline (Tue 23:59 Montreal) | Medium | High | Daily updates on issue #9 keep status visible; WIP = 3 keeps load bounded; scope is limited to designs/mock docs, not a full app | Team Tracker: Day 1 Mon evening / Day 2 Tue morning / Day 3 Tue afternoon; deadline Tue 23:59 | Open |
| R7 | Reminders reach patients after cancellation (cancel/dispatch race) | Medium | High | CA1-CA4 in incident #18: atomic cancellation + pre-send status check + regression tests (issue #6) + DoD verification gate | Incident #18 (Nov 3); `docs/architecture/reminder-scheduler.md` (PR #14); issue #6 | In progress |
