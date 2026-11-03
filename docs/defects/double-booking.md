# Issue #3 — Double Booking Fix

## Root cause

The booking flow checks whether a slot is free and then inserts the booking as two separate steps. When two patients confirm the same slot within ~1 second, both checks pass before either insert commits, so both bookings succeed.

## Fix: SQL sketch

### 1. Partial unique index (database-level backstop)

```sql
CREATE UNIQUE INDEX uq_booking_practitioner_slot
  ON appointments (practitioner_id, slot_start)
  WHERE status IN ('confirmed', 'pending');
```

### 2. Atomic check-and-insert (single transaction)

```sql
BEGIN;
INSERT INTO appointments (patient_id, practitioner_id, slot_start, status)
SELECT $patient_id, $practitioner_id, $slot_start, 'confirmed'
WHERE NOT EXISTS (
  SELECT 1 FROM appointments
  WHERE practitioner_id = $practitioner_id
    AND slot_start = $slot_start
    AND status IN ('confirmed', 'pending')
);
-- 0 rows inserted -> the slot was just taken
COMMIT;
```

The partial index allows the slot to be reused after a cancellation; the transaction makes check-and-insert atomic so two concurrent requests cannot both succeed.

## Test scenario: two patients confirm the same slot at the same time

- Precondition: slot S for practitioner P is available; patients A and B are both on the booking page.
- Steps:
  1. A and B both press Confirm for S within ~1 second.
  2. Both requests reach the server concurrently.
- Expected:
  1. Only one of the two bookings succeeds.
  2. The other patient sees "This slot was just taken" and is shown other available slots.
- Verification: run the two INSERT ... SELECT statements in the same session without a commit between them; exactly one row is created and the loser gets the conflict path.

