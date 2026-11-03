# How the Reminder Scheduling Service Works

Purpose: anyone on the team can debug a reminder problem without depending on one developer.

## Components

| Component | What it does |
|---|---|
| Booking service | Creates, reschedules and cancels appointments in the `appointments` table |
| `reminder_jobs` table | One row per reminder to send: `appointment_id`, `send_at`, `channel` (email/SMS), `status` (`pending`, `sent`, `cancelled`) |
| Reminder worker | Runs every 5 minutes; selects `pending` jobs with `send_at <= now`, marks them `sent`, calls the messaging SDK |
| Messaging SDK | Sends the email or SMS through the provider (see issue #5) |

## Flows

**Booking**
1. Appointment row is created with status `booked`.
2. Two jobs are inserted into `reminder_jobs`: 24 hours before and 2 hours before, status `pending`.

**Reschedule** (issue #1)
1. Appointment time is updated.
2. Existing `pending` jobs for the appointment are set to `cancelled`.
3. Two new `pending` jobs are inserted for the new time.

**Cancellation**
1. Appointment status is set to `cancelled`.
2. A separate step sets that appointment's `pending` jobs to `cancelled`.

**Sending**
1. Worker selects due `pending` jobs.
2. For each job: marks it `sent`, then calls the SDK.
3. The worker does not re-check the appointment's status before sending.

## Where reminder jobs are stored and how they are triggered

Jobs are stored in the `reminder_jobs` table in the main database. Nothing is pushed; the worker polls the table every 5 minutes. A job is sent only if its status is `pending` when the worker reads it.

## Known failure points

| Failure point | What can go wrong |
|---|---|
| Cancellation is two separate writes | If the appointment is cancelled but the job update fails or is delayed, the job stays `pending` and is sent |
| Worker reads before the cancel lands | A job read by the worker in the same window as a cancellation can still be sent |
| Worker does not re-check appointment status | Any job left `pending` for a cancelled appointment is sent with no second check |
| Times stored in UTC, formatted without conversion | Reminders show the wrong local time (issue #4) |
| Provider SDK retirement | All sends fail after the retirement date (issue #5) |
| No automated tests on cancellation | Changes to the cancel flow can break job cleanup without anyone noticing (issue #6) |
