# Issue #1 — Patient Appointment Rescheduling: Design Note and Mock API

**Owner:** Mengqi / Tocky25 (Developer 1)  
**Issue:** [#1](https://github.com/Hz80th/MediLink22/issues/1)  
**Deliverable:** `docs/features/reschedule.md`  
**Status:** Design and mock API only; no production implementation or test execution is claimed.

## Purpose and scope
A patient with an existing `booked` appointment can select a new available slot without cancelling and separately rebooking. The change must free the previous slot, confirm the new date/time, and replace old pending reminders with reminders for the new time. The service must reject a change requested **less than 24 hours before the original appointment**.

This is a simulated service contract, not deployed MediLink code. Appointment and reminder field names are illustrative and can be refined during implementation. No real patient data is used.

## User flow
1. The authenticated patient opens **Appointment details** and selects **Reschedule**.
2. The page checks the appointment belongs to the patient and is `booked`. If the original appointment begins in less than 24 hours, disable rescheduling and show the policy message.
3. The page shows available appointment slots for the same practitioner. The patient chooses a new slot and clicks **Confirm reschedule**.
4. The server rechecks the 24-hour cutoff and slot availability (the list may have become stale); it then commits the appointment change atomically with the booking/slot occupancy change.
5. Old **pending** reminder jobs are cancelled and replacement reminder jobs for the new time are created as part of the same committed change (or by a durable transactional-outbox event); the UI must not report success before consistent state is committed.
6. A success screen displays the new local date/time and the service queues a confirmation email showing the same date/time. Email sending is asynchronous: if delivery fails, record a retry rather than undoing a committed booking.

## Mock request (success case)

`PATCH /api/appointments/A100/reschedule`

```json
{
  "new_slot_id": "S-20261112-1000-DR7",
  "timezone": "America/Toronto"
}
```

**Test fixture:** At a simulated current time of 2026-11-03T12:00:00-05:00, appointment A100 belongs to the authenticated patient, has status `booked`, starts on **2026-11-10 at 15:00 America/Toronto**, and uses practitioner DR7. Slot S-20261112-1000-DR7 is available at **2026-11-12 at 10:00 America/Toronto**.

**Proposed response: `200 OK`**

```json
{
  "appointment_id": "A100",
  "status": "booked",
  "previous_start_at": "2026-11-10T15:00:00-05:00",
  "new_start_at": "2026-11-12T10:00:00-05:00",
  "timezone": "America/Toronto",
  "previous_slot_status": "available",
  "new_slot_status": "booked",
  "reminder_update": "old_pending_cancelled_new_pending_created",
  "confirmation_email": "queued",
  "message": "Your appointment has been rescheduled to November 12, 2026 at 10:00 AM (Toronto time)."
}
```

The confirmation screen displays: **Appointment rescheduled — November 12, 2026, 10:00 AM (Toronto time).** The confirmation email shows the identical new local date/time. Store and transmit instants with explicit timezone offsets/UTC values; convert to clinic local time for patient-facing messages (see Issue #4).

## Reminder synchronization (aligned with Issue #8)
The current reminder design uses a `reminder_jobs` table and a worker polling every five minutes. For bookings, two reminder jobs are scheduled: **24 hours** and **2 hours** before the appointment. In this mock design:

- After a successful reschedule, cancel all **pending** reminder jobs for the old appointment time (do not rewrite already sent messages).
- Insert replacement `pending` jobs for **2026-11-11 10:00** and **2026-11-12 08:00** Toronto time, corresponding to 24 hours and 2 hours before the new appointment.
- Commit appointment/slot/reminder changes consistently. A failed reschedule must leave the original booking and reminder jobs unchanged.
- The existing reminder worker does **not** recheck appointment status immediately before sending, creating a known concurrency risk. A production fix would revalidate job/appointment state and implement safe concurrency controls; this issue only documents that requirement.

## Mock errors and edge cases

| Scenario | Proposed HTTP status | Behavior |
|---|---|---|
| Original appointment starts in **less than 24 hours** | `422 Unprocessable Entity` | Reject: `Rescheduling is not allowed within 24 hours of your appointment. Please contact the clinic.` No booking or reminder changes |
| Original appointment begins **exactly 24 hours** from now | `200 OK` if other conditions pass | Allowed because the issue says **less than** 24 hours (strict cutoff) |
| Another patient just took the selected slot | `409 Conflict` | `This slot is no longer available. Please select another time.` Old booking remains reserved |
| Appointment already cancelled | `409 Conflict` | Reject; do not create reminders |
| Patient not authenticated | `401 Unauthorized` | Reject; do not expose appointment details |
| Patient does not own appointment | `403 Forbidden` (or `404` to prevent information leakage) | Reject; no changes |
| New slot equals existing slot | `422 Unprocessable Entity` | Request a different slot; leave state unchanged |
| Data operation fails halfway through | `500` or retriable service error | Transaction rolled back; original appointment and reminders retained |

**Example late-request mock:**

```json
{
  "error": "RESCHEDULE_CUTOFF",
  "message": "Rescheduling is not allowed within 24 hours of your appointment. Please contact the clinic.",
  "appointment_id": "A100"
}
```

**Example conflict mock:**

```json
{
  "error": "SLOT_UNAVAILABLE",
  "message": "This slot is no longer available. Please select another time."
}
```

## Concurrency and transactional behavior (design sketch)
- On confirmation, check that the old appointment is still `booked`, the patient is authorized, and the cutoff has not been reached.
- Lock or otherwise atomically claim the new practitioner/time slot (aligned with the unique-slot constraint in Issue #3). If the slot was taken, return `409`.
- Update the existing appointment to the new slot **within the same transaction** as releasing the old slot and updating pending reminder jobs. Commit before returning success.
- Two simultaneous reschedule attempts must not claim the same slot. Exactly one may succeed; the other receives a conflict.
- Queue confirmation email after durable commit, preferably using an outbox for retries; do not assert successful delivery when only queued.

## Acceptance criteria traceability and review checks

| AC | Evidence in this note | Reviewer check |
|---|---|---|
| AC1: choose a new available slot from details | User flow steps 1–4, PATCH example | New slot is shown and revalidated before commit |
| AC2: release original slot upon confirmation | Success response and transactional behavior | Old slot remains occupied on failure; available immediately after commit |
| AC3: new date/time on confirmation screen and email | Success response and user-facing message | Screen/email agree, correct local time, email delivery distinguished from queueing |
| AC4: remove old reminders and schedule new ones | Reminder synchronization section | Old pending jobs cancelled; new 24-hour/2-hour jobs created |
| AC5: block within 24 hours with clear message | Mock errors and cutoff boundary | `<24h` blocked, `=24h` allowed when otherwise valid |

## Design verification checklist (not automated test results)
- [ ] Peer reviews the note and checks it against Issue #1 AC1–AC5.
- [ ] Verify mock JSON parses and example dates/times are internally consistent.
- [ ] Confirm consistency with `docs/architecture/reminder-scheduler.md` and Issue #3 concurrent booking constraints.
- [ ] Confirm no TODOs or contradictory assumptions remain.
- [ ] Record reviewer feedback and owner's independent verification before merging.

## Assumptions and limits
- Product policy is enforced on the **original** appointment's start time; rescheduling to a new time that is too soon should be clarified with the product owner. This example uses a future slot beyond 24 hours.
- The example shows an email **queued**, not delivered. A real sending/verification mechanism is outside this cycle's deliverable.
- The design does not implement the API, slot database, worker, emails, or executable tests; those need separate follow-up implementation work.
