# Production Support Runbook

Purpose: route production-support requests through one person per day, so the rest of the team can finish planned work without random interruptions.

## Severity levels

| Severity | Definition | Example | Handled by | First response |
|---|---|---|---|---|
| Critical | Patients cannot book, or patients receive wrong or harmful information | Cancelled patients still receive reminders; booking page down | Support person of the day + EM, immediately | 15 minutes |
| High | A feature is broken but there is a workaround | Reminders show the time in UTC; admin cannot block time | Support person of the day | 2 hours (working hours) |
| Normal | Cosmetic issue, question, or small request | Typo in an email template; clinic asks how to export a list | Added to Backlog, prioritized at the next planning | Next working day |

## Daily support rotation

| Day | Support person | Backup |
|---|---|---|
| Monday | Tetant | Mengqi |
| Tuesday | Shenhuan | Tetant |
| Wednesday | Mengqi | Tetant |
| Thursday | Tetant | Shenhuan |
| Friday | Shenhuan | Mengqi |

Only the support person of the day is interrupted. Everyone else keeps their In Progress card.
The support person's own planned work is expected to slow down that day, and that is accepted, not counted against them.

## How a request is handled

1. The request is opened as a GitHub issue with the label `type: defect` or `type: maintenance` and a severity in the title, for example `[Critical] Cancelled patients still get reminders`.
2. The support person of the day assigns it to themselves and replies within the first-response time above.
3. **Critical:** start a sync call immediately (see [communication.md](communication.md)). The EM decides whether new work pauses. The issue stays the written record.
4. **High:** the support person works on it or hands it to the issue owner with a written note. No call needed unless it is blocked.
5. **Normal:** moved to Backlog; nobody stops current work.
6. After any Critical incident, an RCA is written in the issue (Five Whys, at least two corrective or preventive actions, no blame on a person).

## Escalation

If the support person cannot resolve a Critical issue within 1 hour, they call the backup and the EM.
