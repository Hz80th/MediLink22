# Issue #20 — CA3: Cancellation Reminder Regression Tests

**Owner:** Mengqi / @Tocky25  
**Incident:** [#18](https://github.com/Hz80th/MediLink22/issues/18)  
**Dependencies:** [#6 — Original cancellation test plan](https://github.com/Hz80th/MediLink22/issues/6), [#19 — Pre-send guard design](https://github.com/Hz80th/MediLink22/issues/19), [#8 — Scheduler architecture](https://github.com/Hz80th/MediLink22/issues/8)  
**Deliverable:** `docs/tests/cancellation-regression.md`  
**Status:** Test specification for a simulated service; **no tests have been executed** and no production fix is asserted.

## 1. Purpose and test scope

Prevent regression of the Critical incident in which cancelled patients still received appointment reminders. This plan extends `docs/tests/cancellation-tests.md` and adds checks for the proposed guard in `docs/defects/reminder-pre-send-guard.md`.

The existing documented scheduler polls `reminder_jobs` every five minutes, uses `pending`, `sent`, `cancelled` states, and does **not** re-check appointment status. CA2 proposes a status check, atomic claim, and coordination with cancellation. `claimed` and `failed` are **proposed** job states, not asserted to be implemented.

Tests distinguish **design expectations** from observed execution. Until a mock harness or implementation is run, all cases remain **Not Run**.

## 2. Reproducible fixture and observable outputs

- **Frozen clock:** 2026-11-03 12:00:00 `America/Toronto`.
- **Fictional IDs only:** `A301`–`A309` appointments and corresponding `R301`–`R309` reminders.
- **Appointment fields:** `id`, `status` (`booked`/`cancelled`), `start_at`, `version`.
- **Reminder fields:** `id`, `appointment_id`, `status` (`pending`, `cancelled`, plus proposed `claimed`/`failed`/`sent`), `send_at`, `schedule_version`.
- **Fake provider:** records each attempted send as `(reminder_id, timestamp, idempotency_key)`; it can be configured to accept or fail an attempt.
- **Fake worker:** explicitly invoked after frozen-clock advancement (do not wait in real time).
- **Interleaving hooks:** `before_claim`, `after_claim`, `before_provider_send` let a test trigger cancellation at a precise boundary.
- **Assertions:** inspect appointment record, reminder record, provider call log, and event ordering. Do not infer actual message delivery from queued/accepted state.

**Important policy assumption:** A cancellation within 24 hours is allowed for this simulation unless the product owner specifies otherwise. Issue #1's *rescheduling* cutoff does not automatically establish a cancellation cutoff.

## 3. Test cases — input, steps and expected results

### RT-01 — Normal future appointment cancellation
**Input:** `A301.status=booked`; `R301.status=pending`; appointment starts Nov 10 at 15:00 Toronto time; reminder is not due.  
**Steps:** (1) Request cancellation of A301. (2) Commit cancellation. (3) Read both records. (4) Advance fake clock past reminder due time and run worker.  
**Expected:** `A301.status=cancelled`; `R301.status=cancelled` or otherwise ineligible; provider sends for R301 = **0**.

### RT-02 — Repeat cancellation (idempotency)
**Input:** `A302.status=cancelled`; `R302.status=cancelled`.  
**Steps:** (1) Request cancellation again. (2) Read records and provider log.  
**Expected:** Stable already-cancelled/idempotent response; appointment stays cancelled; no new reminder jobs; sends = **0**. Exact HTTP response remains subject to product contract.

### RT-03 — Cleanup of multiple pending reminders
**Input:** `A303.status=booked`; two pending reminder jobs for the same appointment (24h and 2h).  
**Steps:** (1) Cancel A303. (2) Inspect all jobs linked to A303. (3) Run worker after each planned trigger.  
**Expected:** All unsent jobs are invalidated; no pending job remains eligible; sends = **0**.

### RT-04 — Stale pending job after partial write failure
**Input:** `A304.status=cancelled`; `R304.status=pending` due now, simulating failure of the separate reminder invalidation write in the old architecture.  
**Steps:** (1) Run worker against the CA2 proposed pre-send guard. (2) Inspect event log.  
**Expected:** Worker reads the cancelled appointment, suppresses R304, records `appointment_cancelled`, provider sends = **0**. Demonstrates defense in depth even if CA1 has not repaired the stale job.

### RT-05 — Cancellation wins race before worker claim
**Input:** `A305.status=booked`; `R305.status=pending` and due.  
**Steps:** (1) Pause worker at `before_claim`. (2) Commit cancellation. (3) Resume worker.  
**Expected:** Worker cannot make a valid claim; R305 is suppressed; sends = **0**. Log ordering proves cancellation committed before claim.

### RT-06 — Worker claims before cancellation
**Input:** `A306.status=booked`; `R306.status=pending` and due.  
**Steps:** (1) Pause worker at `after_claim`. (2) Initiate cancellation. (3) Resume both according to the CA1/CA2 synchronization rule. (4) Record cancellation acknowledgment and provider send timestamps.  
**Expected:** Behavior follows an **explicitly agreed in-flight policy**: either the cancellation waits for an already-committed dispatch boundary, or dispatch is suppressed before cancellation acknowledges. The test must establish the ordering. **Do not claim zero post-cancellation deliveries without implementing and verifying that guarantee.**

### RT-07 — Two workers competing for the same reminder
**Input:** `A307.status=booked`; `R307.status=pending` and due; two worker instances.  
**Steps:** (1) Start both at `before_claim`. (2) Let both attempt conditional claim. (3) Inspect claims/provider calls.  
**Expected:** Exactly **one** successful claim; at most **one** provider request (assuming the configured provider idempotency contract).

### RT-08 — Already-cancelled reminder cannot dispatch
**Input:** `A308.status=booked`; `R308.status=cancelled`, due.  
**Steps:** Run worker and inspect logs.  
**Expected:** No claim, no dispatch, sends = **0**.

### RT-09 — Messaging provider failure
**Input:** `A309.status=booked`; `R309.status=pending`, due; fake provider fails.  
**Steps:** (1) Claim and attempt send. (2) Inspect persisted job status and retry metadata.  
**Expected:** Job is **not falsely marked `sent`**; failure recorded and retry behavior respects idempotency. Actual retry strategy is a follow-up implementation detail.

## 4. Traceability

| Requirement | Regression cases |
|---|---|
| Issue #20: normal cancellation | RT-01 |
| Already-cancelled appointment | RT-02 |
| Pending reminders invalidated | RT-01, RT-03 |
| Pre-send suppression for cancelled appointments | RT-04, RT-05 |
| Cancellation/dispatch race condition | RT-05, RT-06 |
| Input / expected / actual / execution tracking | Sections 2, 3, 5 |
| DoD verification references | Section 6 |
| CA2 guard extra protections | RT-07, RT-08, RT-09 |
| Original Issue #6: cancellation reminder behavior | RT-01, RT-02, RT-03, RT-04, RT-06 |

## 5. Execution record (must be populated only after running)

| Case | Status | Actual result | Evidence / log or PR | Tester / time |
|---|---|---|---|---|
| RT-01 | Not Run | — | — | — |
| RT-02 | Not Run | — | — | — |
| RT-03 | Not Run | — | — | — |
| RT-04 | Not Run | — | — | — |
| RT-05 | Not Run | — | — | — |
| RT-06 | Not Run | — | — | — |
| RT-07 | Not Run | — | — | — |
| RT-08 | Not Run | — | — | — |
| RT-09 | Not Run | — | — | — |

**Execution rule:** A test can be marked `Pass` only with a runnable harness/implementation, the actual observed outcome, and evidence. `Not Run` is acceptable as an honest *test-design* status, but **does not meet a requirement to demonstrate executed automated tests**. Any claim that automated regression coverage exists requires separate executable test code and test-run output.

## 6. Definition of Done / Verification linkage

For any future PR changing cancellation or reminder dispatch, include a section like:

```markdown
### Cancellation reminder verification
Reference: docs/tests/cancellation-regression.md
Applicable case IDs: RT-01, RT-03, RT-04, RT-05, RT-06
Execution environment / commit:
Actual outcomes and evidence:
Reviewer:
```

The PR owner must select applicable cases and record the *actual* results or explicitly mark them `Not Run` with an explanation. `docs/dod.md` already points to `docs/tests/cancellation-tests.md`; this document extends that reference for CA3. The team should decide whether to add this second path directly to the DoD as part of CA4.

### Checklist for this design-document PR
- [ ] Link PR to #20 with `Closes #20`.
- [ ] Review traceability against every Issue #20 acceptance criterion.
- [ ] Confirm examples agree with CA2 design and existing scheduler architecture.
- [ ] Independent reviewer approval received.
- [ ] Resolve review feedback and perform owner verification after approval.
- [ ] Do not mark executable tests Passed until there are real execution logs.

## 7. Follow-ups and limitations

- **CA1:** Shenhuan's consistency/transaction design must specify how pending reminder invalidation is committed.
- **CA2 / Issue #19:** The dispatch/cancel synchronization contract must define what happens when a send has already started.
- **CA3 / Issue #20:** These scenarios are reproducible specifications, not actual automated test scripts. An executable in-memory/mock harness or production test framework is necessary to prove the behavior.
- **Critical incident #18:** Actual cause remains preliminary until correlated cancellation, worker, and provider evidence is available.
- All records, times and IDs in this file are fictional test fixtures.
