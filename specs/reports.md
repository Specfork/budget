# Reports

**Status:** Draft · **Prefix:** RPT

## Purpose

Read-only summaries of the person's own data: where money went, what came in and out, and what
they are worth over time.

## Requirements

| ID | Requirement | Verified by |
|---|---|---|
| RPT-1 | Every report takes a date range (default last 6 whole months to today), `interval`: `day`\|`week`\|`month` (default)\|`year`, and optional filters using [RULE-2](rules.md) conditions; weeks start on the file's first day of week. All sums use live, non-parent transactions. Any report table exports to CSV with the same rows and columns; amounts as decimals per [FILE-11](files.md). | range Jan–Jun, monthly → 6 columns; export spending → CSV row per category |
| RPT-2 | Net worth: per interval end `e`, `net_worth(e)` = Σ `amount` of transactions in all accounts (on and off budget, open and closed) dated ≤ `e`, also per account; filterable by account. Transfers between included accounts cancel out. | on 700 + off 5000 at Jan 31 → 5700 |
| RPT-3 | Cash flow (on-budget accounts): per interval `income` = Σ positive amounts and `expense` = Σ negative amounts, both excluding transfers; `transfers` = Σ transfers to or from off-budget accounts, shown apart; `balance` at interval end. End date is capped at today. | +3000, −1200, −500 to off-budget → income 3000, expense −1200, transfers −500 |
| RPT-4 | Spending (on-budget accounts, expense categories): per category and group, Σ `activity` ([BUD-3](budget.md)) per interval, plus "uncategorised" for rows without category; compares a chosen month with the average of the previous 3, 6 or 12 months and with that month's `assigned`. | food Jan −600, assigned 500 → over by 100 |
| RPT-5 | Custom report: `group_by`: `category`\|`category_group`\|`payee`\|`account`\|`tag`\|`interval`; `measure`: `payments` (Σ amounts < 0)\|`deposits` (Σ > 0)\|`net`\|`assigned`; `chart`: `table`\|`bar`\|`line`\|`area`\|`donut`; `show_off_budget` (default false), `show_hidden` (default true), `show_uncategorised` (default true); sort by `amount_asc`\|`amount_desc`\|`name`. Uncategorised buckets in custom reports: `transfer` (transfer rows), `off_budget` (rows in off-budget accounts), `other` (no category). | group by payee, payments, Jan → one row per payee; off-budget row shown under `off_budget` |
| RPT-6 | Saved reports: `id`, `name` (unique, 1–100 chars), all RPT-5 settings and filters, `date_range` fixed or relative (`this_month`, `last_n_months` N 1–60, `year_to_date`, `last_year`); create, rename, update, delete. A dashboard holds report tiles in a grid. | save relative "last 3 months" → next month the range shifts by one |
| RPT-7 | Reports show only arithmetic over the person's data: no advice, scores, product offers or predictions of returns. | manual: no recommendation text anywhere |

## Out of scope

- Forecasts, investment performance, tax reports.
- Shared or public report links.

## Decisions

- Net worth includes closed accounts so past months stay correct after closing.
