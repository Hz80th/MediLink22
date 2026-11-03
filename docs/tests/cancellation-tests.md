# Issue #6 — Appointment Cancellation Test Plan

**Owner:** Mengqi (Developer 1; assignment pending)  
**Issue:** #6  
**Status:** Test specifications only — execution not yet performed  
**Scope:** Simulated MediLink release, without a full application.

## Assumptions
For the simulation, an appointment has `id`, `start_time` and `status` (`booked` or `cancelled`); its unsent reminder has `scheduled` or `cancelled` status. Successful cancellation changes the appointment status and invalidates any unsent reminder. Because the product policy is unspecified, cancellation within 24 hours is **assumed allowed** and must be confirmed. Use a frozen clock: **2026-11-03 12:00 America/Toronto**. All data is fictional.

## Test cases

| ID | Input / precondition | Steps | Expected result |
|---|---|---|---|
| CT-01 | A100 booked for Nov 10, 2026 at 15:00 local; reminder R100 scheduled | Cancel A100 once; check appointment and reminder state | Cancellation succeeds; A100 is `cancelled`; R100 becomes inactive |
| CT-02 | A101 booked for Nov 4, 2026 at 08:00 local (20 hours away); reminder R101 scheduled | Cancel A101; inspect records | Under assumed policy, cancellation succeeds and reminder is inactive; revise if late cancellation is restricted |
| CT-03 | A102 is already `cancelled`, with no active reminder | Request cancellation a second time; inspect records | Already-cancelled/idempotent response; status unchanged, no duplicate effects, no new reminder |
| CT-04 | A103 booked; reminder R103 queued but not yet sent | Cancel A103; advance fake clock past reminder trigger; run simulated reminder dispatcher; inspect outbound log | Zero reminder messages for A103 after successful cancellation |
| CT-05 | A104 booked; reminder dispatcher might race with cancellation | Simulate reminder dispatch occurring around cancellation; require status check immediately before sending | No reminder sent **after cancellation commits**; document whether a reminder was sent beforehand |

## Verification and traceability

1. Prepare fictional appointment and reminder records for each case.
2. Execute each case in a mock or test harness when available. Inspect appointment status, reminder status and outbound send log.
3. For each run, record date, tester, actual result, Pass/Fail and screenshot/log reference.
4. A test design is **not** a passing automated test. Until execution is available, mark **Not Run**.
5. Every cancellation-related PR must include a **Verification** section linking to `docs/tests/cancellation-tests.md`, naming the applicable CT case IDs, and recording outcomes before Done.

| Case | Execution status | Actual result | Evidence |
|---|---|---|---|
| CT-01 | Not Run | — | — |
| CT-02 | Not Run | — | — |
| CT-03 | Not Run | — | — |
| CT-04 | Not Run | — | — |
| CT-05 | Not Run | — | — |

## Acceptance criteria mapping

- **AC1:** CT-01 (future cancellation), CT-02 (within 24 hours), CT-03 (already cancelled).
- **AC2:** CT-01 and CT-04 (scheduled-reminder invalidation), CT-05 (race-condition safeguard).
- **AC3:** Inputs, steps and expected outputs documented for each case.
- **AC4:** Verification process above explicitly requires linking this plan from future cancellation-related PRs.

## DoD review checklist

- [ ] PR linked to Issue #6
- [ ] Acceptance criteria checked against this file
- [ ] Reviewed by a teammate who is not the author
- [ ] Review comments resolved
- [ ] Documentation self-checked
- [ ] Verification recorded before merge
- [ ] For cancellation-related work, verify the applicable test cases in `docs/tests/cancellation-tests.md` and record the results before marking the issue Done.

## Open questions / limits

- Is cancellation less than 24 hours before an appointment allowed?
- Does repeating a cancellation return success or an `already cancelled` response?
- Is reminder delivery guaranteed to re-check current appointment status before sending?
- No automated test code or actual run results are claimed in this simulation document; implementing those requires an executable test harness.
