# Schedules

**Status:** Draft · **Prefix:** SCH

## Purpose

Bills, paydays and other recurring transactions: when they are due, whether they happened, and
optionally posting them automatically.

## Requirements

| ID | Requirement | Verified by |
|---|---|---|
| SCH-1 | Schedule: `id`, `name?` (trimmed, ≤ 100 chars, unique among schedules when set, else `name_taken`), `account?`, `payee?`, `amount` with `amount_op`: `is`\|`is_approx`\|`between` (default `is_approx`), `date_op`: `is`\|`is_approx` (default `is_approx`), `recurrence` or a single `date`, `auto_post` (default false), `completed` (default false), `next_date`, `actions` (set category, notes, split per [RULE-7](rules.md)). A date is required, else `invalid_schedule`. | schedule without date → `invalid_schedule` |
| SCH-2 | Recurrence: `start` date, `frequency`: `daily`\|`weekly`\|`monthly`\|`yearly`, `interval` 1–999 (default 1), `end`: `never` (default)\|`after` N (1–999 occurrences)\|`until` date. | weekly, interval 2, start Mon 5 Jan → 5, 19 Jan, 2 Feb |
| SCH-3 | Monthly patterns (optional, replace the start's day): list of `day n` (1–31 or −1 = last day) and `weekday nth` (`mon`…`sun`, nth 1–5 or −1 = last). A `day n` past the month's end falls on its last day. | monthly day 31 → 31 Jan, 28 Feb, 31 Mar (2027) |
| SCH-4 | Weekend: `weekend_shift`: `none` (default)\|`before`\|`after`; a Saturday or Sunday occurrence moves to the previous Friday or next Monday. | 1st on Saturday, `after` → Monday the 3rd |
| SCH-5 | `next_date` = first occurrence (after shift) on or after today that is not yet paid or skipped; when the recurrence has ended, the schedule becomes `completed`. A one-off schedule is `completed` once paid. | one-off paid → completed |
| SCH-6 | Matching: a new, imported or fed transaction matches a schedule when account (if set) and payee (if set) are equal, the amount passes `amount_op` ([RULE-4](rules.md)), and its date is within ±2 days of an unpaid occurrence (`is_approx`) or on it (`is`). A match sets `schedule_id`, applies the schedule's actions (filling empty fields) and marks that occurrence paid. | due 1 Mar −120000, approx; import 2 Mar −121000 → linked |
| SCH-7 | Status, first that holds: `completed`; `paid` (current occurrence linked); `due` (`next_date` = today); `upcoming` (within the upcoming window); `missed` (`next_date` < today); else `scheduled`. | next_date yesterday, unpaid → `missed` |
| SCH-8 | Upcoming window: file setting, default 7 days; choices 1, 7, 14 days, 1 month, rest of current month, or custom N days/weeks/months/years (N 1–99). Upcoming list: occurrences of all non-completed schedules from today to the end of the window; the account view shows them as uncommitted rows that are not in any balance. | window 14, next_date in 10 days → `upcoming`; rent due in 3 days → shown, balance unchanged |
| SCH-9 | Auto-post: once per day (on open and at the first change of date while open), for each `auto_post` schedule with an unpaid occurrence ≤ today, create the transaction (account, payee, amount (for `between`, the lower bound by absolute value), actions, `cleared` = false, dated the occurrence) and mark it paid; repeat for every missed occurrence. Needs `account`, else not auto-posted and flagged. Skip next: marks the current occurrence skipped and advances `next_date`. Post now: creates the transaction for `next_date` dated today and marks it paid. | monthly 1st, app closed Jan–Mar, open 5 Mar → 3 rows (1 Jan, 1 Feb, 1 Mar); skip on 1 Feb → next_date 1 Mar |

## Out of scope

- Balance forecasting from schedules; schedule-funded budget goals.
- Business-day calendars beyond weekends.

## Decisions

- A day past the month's end clamps to the last day; the source skipped those months.
- Paid occurrences are tracked per occurrence; the source inferred them from linked dates.
