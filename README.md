---
specfork: 1
name: "@specfork/budget"
summary: Local-first personal envelope budget with accounts, split and transfer transactions, monthly assign and rollover, schedules, rules, reconciliation, file import, bank feeds, reports and optional sync.
license: CC-BY-4.0
requires:
  "@specfork/auth": "^0.3.0"
  "@specfork/privacy": "^0.1.0"
derived_from:
  repo: https://github.com/actualbudget/actual
  commit: d1a536f35c5a114eda0582d5a1d744a9103cc90e
  url: https://actualbudget.org
  license: MIT
---

# Budget

One person plans and tracks their own money with envelope budgeting: every unit of income is
given a job in a category, spending draws envelopes down, and what is left (or overspent) rolls
into next month by fixed rules. Data lives on the person's own devices; a sync server is optional.
Behaviour derived from Actual Budget's open-source code (MIT), written fresh and stack-free.
Not affiliated with or endorsed by the Actual Budget project or its contributors. The name is used
only to credit the source.

| Area | Prefix | Covers |
|---|---|---|
| [files.md](specs/files.md) | FILE | Budget files, currency, money and rounding, local-first, export, import, inventory |
| [accounts.md](specs/accounts.md) | ACC | On/off-budget accounts, balances, opening balance, close/reopen/delete, reconciliation |
| [transactions.md](specs/transactions.md) | TXN | Fields, ordering, cleared/reconciled, split transactions, transfers |
| [categories.md](specs/categories.md) | CAT | Payees, category groups, categories, merge and delete, notes, tags |
| [budget.md](specs/budget.md) | BUD | Monthly assign, activity, available, rollover, overspending, to-budget, hold, budget actions |
| [rules.md](specs/rules.md) | RULE | Conditions, actions, stages and order, split actions, category learning |
| [schedules.md](specs/schedules.md) | SCH | Recurring transactions, recurrence, matching, status, auto-posting |
| [import.md](specs/import.md) | IMP | CSV, OFX and QIF import, duplicate matching, generic bank feed |
| [reports.md](specs/reports.md) | RPT | Spending, cash flow, net worth, custom reports |
| [sync.md](specs/sync.md) | SYNC | Optional sync server, change log, conflict rule, end-to-end encryption, server delete-all |

Single user: a budget file belongs to one person and has no members, roles or sharing; a person may
keep several budget files. `@specfork/auth` is used only by the optional sync server, in `single_owner`
mode ([SYNC](specs/sync.md)); export and import follow `@specfork/privacy` ([FILE](specs/files.md)).
Money is integer minor units ([FILE-5](specs/files.md)); dates are calendar dates `YYYY-MM-DD`
without time zone; months are `YYYY-MM`; names are `snake_case`. The app gives no financial
advice: it shows arithmetic over the person's own data only.
