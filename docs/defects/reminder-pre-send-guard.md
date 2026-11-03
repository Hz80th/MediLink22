# Issue #19 — CA2: Reminder Pre-send Guard (Design and Mock Verification)

**Owner:** Mengqi / @Tocky25  
**Related:** [Critical incident #18](https://github.com/Hz80th/MediLink22/issues/18), [cancellation tests #6](https://github.com/Hz80th/MediLink22/issues/6), [reminder scheduler #8](https://github.com/Hz80th/MediLink22/issues/8), [CA3 #20](https://github.com/Hz80th/MediLink22/issues/20)  
**Deliverable:** `docs/defects/reminder-pre-send-guard.md`  
**Status:** Design and mock scenarios only. No deployed code, production repair, or automated test execution is claimed.

## 1. Problem and evidence

The current reminder worker polls `reminder_jobs` every five minutes, selects jobs with status `pending` whose `send_at` is due, marks a job `sent`, and calls the messaging SDK. It does **not** re-check the appointment status before sending. Also, appointment cancellation and pending-job cancellation are separate database writes. Therefore, pending or already-claimed reminders can be sent after the appointment is cancelled.

The architecture above is documented in `docs/architecture/reminder-scheduler.md` (Issue #8). Incident #18 contains a **preliminary** Five Whys; actual failure timing and root cause await support investigation. No logs or production measurements are invented here.

## 2. Proposed pre-send validation

For each due reminder candidate:

1. Read the reminder's current status and related appointment status from authoritative storage, not a stale UI or cached snapshot.
2. Check both `appointment.status == 'booked'` and `reminder.status == 'pending'` (or another documented eligible state), as well as an up-to-date reminder/appointment version or schedule identity.
3. If the appointment is `cancelled`, or the reminder is cancelled or superseded, mark/suppress the reminder without calling the messaging SDK. Record a reason such as `appointment_cancelled`.
4. If both are valid, atomically **claim** the reminder for dispatch, preventing two workers from sending the same job. Use a conditional update or transaction with row locks, depending on the eventual database.
5. Dispatch using an idempotency key where the messaging provider supports one; record the provider result and retry failures safely. Do **not** mark a job `sent` before verifying provider acceptance.
6. Record a timestamped outcome for audit and CA3 regression tests.

### Suggested mock states

| Entity | Relevant states |
|---|---|
| Appointment | `booked`, `cancelled` |
| Reminder job | `pending`, `claimed`, `cancelled`, `sent`, `failed` |

`claimed` and `failed` are **proposed extensions**; they are not asserted to exist in the current architecture.

## 3. Race condition: cancellation vs dispatch

A status read alone is **not sufficient**: a patient could cancel immediately after the read but before the provider call. The implementation therefore requires an explicit consistency contract and serialization strategy.

**Design proposal:** A transaction takes a lock or equivalent serialization guard covering the appointment and relevant reminder claim. Cancellation uses the same coordination mechanism to mark the appointment cancelled and invalidate unclaimed pending jobs. A worker may claim a job only while the appointment is still booked. Claims include a unique job ID/version to reject stale or duplicate sends. Re-checking status before the external call minimizes the window.

**Important limit:** A database transaction cannot atomically undo an SMS/email that has already been accepted by an external provider. If cancellation occurs after a send has irrevocably started, the system must distinguish that case in logs and product policy. To guarantee *no messages after cancellation confirmation*, cancellation confirmation must coordinate with in-flight dispatch (e.g., hold/serialize dispatch or await a defined send boundary), and the delivery semantics must be specified and tested. The mock design does not claim to prove that guarantee.

### Ordering scenarios

- **Cancellation wins:** Cancel commits first. Worker sees `cancelled`, suppresses reminder, zero provider calls.
- **Worker observes cancellation before claim:** Worker rejects claim and suppresses reminder.
- **Worker claims first, cancellation races:** State machine coordinates an in-flight claim with cancellation; the implementation must either prevent a send after cancellation acknowledgment or explicitly record a send already in progress. Do not report unconditional prevention without verifying the integration.
- **Duplicate workers:** Only one succeeds in claiming the same job; no duplicate provider sends.
- **Messaging SDK failure:** Record `failed` or retryable outcome; do not incorrectly mark `sent`.

## 4. Pseudocode (illustrative, not executable application code)

```text
for each due reminder candidate:
    begin database transaction
      lock appointment and reminder rows in a consistent order
      if appointment.status != BOOKED:
          cancel/suppress pending reminder; record reason
          commit; continue
      if reminder.status != PENDING or reminder.schedule_version != appointment.version:
          record skip; commit; continue
      atomically mark reminder CLAIMED with a unique claim token
    commit

    coordinate in-flight claim with cancellation policy
    revalidate eligible state at dispatch boundary
    if no longer valid: cancel/suppress claim; do not send
    else:
        call provider using an idempotency key (if supported)
        record accepted / failed outcome, respecting retry policy
```

The coordination and dispatch boundary are design requirements still to be defined and implemented, not implied guarantees of this pseudocode.

## 5. Mock scenarios and expected outcomes

| ID | Input / initial state | Simulated steps | Expected outcome |
|---|---|---|---|
| PG-01 | A200 `booked`, R200 `pending`, due | Worker validates and claims R200; no cancellation | One eligible dispatch; successful provider acceptance recorded as `sent` |
| PG-02 | A201 `cancelled`, R201 still `pending`, due | Worker loads A201 before attempting send | R201 suppressed/cancelled; provider send count = 0 |
| PG-03 | A202 `booked`, R202 `cancelled` | Worker polls R202 | No claim and no send |
| PG-04 | A203 is cancelled while worker is preparing to claim R203 | Apply cancellation/claim serialization | If cancel commits first, no send; if send was already initiated, log ordering and apply documented in-flight policy |
| PG-05 | Two workers claim the same due R204 | Interleave claim operations | Exactly one claim succeeds; at most one provider request under normal idempotent dispatch |
| PG-06 | Provider rejects a send for valid A205 | Trigger provider failure | Do not mark job `sent`; record error/retry path |

These are **designed test cases**. They are not executed results.

## 6. Example mock decision responses

**Cancelled appointment (suppressed):**

```json
{
  "appointment_id": "A201",
  "reminder_id": "R201",
  "appointment_status": "cancelled",
  "reminder_status": "cancelled",
  "dispatch_decision": "suppressed",
  "reason": "appointment_cancelled",
  "provider_send_count": 0
}
```

**Valid booked appointment (eligible):**

```json
{
  "appointment_id": "A200",
  "reminder_id": "R200",
  "appointment_status": "booked",
  "dispatch_decision": "eligible_to_send",
  "provider_send_count": null,
  "note": "Final count and sent status require execution and provider result"
}
```

## 7. Acceptance criteria mapping

| Issue #19 acceptance criterion | Evidence in this document |
|---|---|
| Worker checks appointment status before sending | Sections 2 and 4 |
| Suppress reminders for cancelled appointments | Sections 2, 5 (PG-02), 6 |
| Address cancellation/dispatch races in design | Section 3; PG-04 and PG-05 |
| Mock scenarios show allowed and blocked reminders | PG-01/PG-02, JSON examples |
| Document limits and verification results | Sections 3 and 8 |

## 8. Verification and Definition of Done

| Check | Status | Evidence / next step |
|---|---|---|
| Design covers Issue #19 AC1–AC5 | Prepared for reviewer check | Sections 2–7 |
| Mock scenarios internally consistent | Self-check pending | PG-01–PG-06 |
| Executable regression tests passed | **Not run** | Requires test harness; follow-up Issue #20 |
| Incident root cause confirmed with actual logs | **Not confirmed** | Support findings in Issue #18 |
| Independent PR reviewer approved | **Pending** | PR review |
| Owner performed verification after approval | **Pending** | Before merge |

Before merging: link PR to #19 (`Closes #19`), copy and complete the repository's `docs/dod.md` checklist, ensure no unresolved review threads, and record a genuine post-review verification. Changes to production behavior are **out of scope** for this document-only mock deliverable.

## 9. Follow-up dependencies

- **CA1 (Shenhuan):** define atomic/consistent cancellation and reminder invalidation; align locking/ordering with this design.
- **CA3 / Issue #20 (Mengqi):** turn PG-01–PG-06 into reproducible tests, including cancellation/send races, and record actual outcomes where executable tooling exists.
- **CA4 (Tetant):** apply the DoD gate to cancellation/reminder changes.
- **Support investigation:** update Incident #18 when observed evidence identifies the actual triggering path.
