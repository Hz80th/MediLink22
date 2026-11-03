# Issue #3 — Double Booking Fix

## Root cause

The booking flow checks whether a slot is free and then inserts the booking
as two separate steps. When two patients confirm the same slot within ~1
second, both checks pass before either insert commits, so both bookings
succeed.

## Fix: SQL sketch

### 1. Partial unique index — the guarantee

```sql
CREATE UNIQUE INDEX uq_booking_practitioner_slot
  ON appointments (practitioner_id, slot_start)
  WHERE status = 'booked';
```

This is what actually blocks the second booking: no matter how the two
requests interleave, the database cannot accept a second `booked` row for
the same practitioner and slot.

### 2. Atomic check-and-insert — the friendly path

```sql
BEGIN;
INSERT INTO appointments (patient_id, practitioner_id, slot_start, status)
SELECT $patient_id, $practitioner_id, $slot_start, 'booked'
WHERE NOT EXISTS (
  SELECT 1 FROM appointments
  WHERE practitioner_id = $practitioner_id
    AND slot_start = $slot_start
    AND status = 'booked'
);
-- 0 rows inserted -> the slot was just taken
COMMIT;
```

The `WHERE NOT EXISTS` check is not atomic on its own: two concurrent
requests can both pass it. The unique index is what rejects the second
booking. The transaction exists so the normal losing path returns 0 rows
and the app shows "This slot was just taken" directly; if a unique-violation
error surfaces instead, the app handles it the same way. The partial index
also lets the slot be reused after a cancellation.

## Test scenario: two patients confirm the same slot at the same time

- Precondition: slot S for practitioner P is available; patients A and B are
  both on the booking page.
- Steps:
  1. A and B both press Confirm for S within ~1 second.
  2. Both requests reach the server concurrently.
- Expected:
  1. Only one of the two bookings succeeds (the unique index rejects the
     second active row).
  2. The other patient sees "This slot was just taken" and is shown other
     available slots.
- Verification: run the two inserts from **two separate sessions at the
  same time** (one session cannot test concurrency). Exactly one insert
  succeeds; the loser either inserts 0 rows or hits the unique index, and
  in both cases the app shows "This slot was just taken".
