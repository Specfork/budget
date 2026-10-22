# Files

**Status:** Draft · **Prefix:** FILE

## Purpose

Budget files, the one currency each file uses, how money is stored and rounded, offline use, and
what export, import and the data inventory cover.

## Requirements

| ID | Requirement | Verified by |
|---|---|---|
| FILE-1 | A budget file holds all data of one budget: accounts, transactions, payees, categories, budget months, rules, schedules, reports, notes, tags and preferences. Fields: `id` (random, immutable), `name` (trimmed, 1–100 chars, unique case-insensitively among the person's files on that device or server), `currency_code`, `created_at`. One person may keep any number of files; exactly one is open at a time. Empty or taken name → `invalid_name`. | create "Home" twice → second `invalid_name` |
| FILE-2 | Create: `name` defaults to "Budget", then "Budget 2", "Budget 3"… (first free). A new file has one income group `income` holding the categories `income` and `starting_balances`, one expense group `expenses` with no categories, no accounts and no transactions. | create two files without names → "Budget", "Budget 2" |
| FILE-3 | Operations: list, create, open, close, rename, duplicate, delete. Duplicate copies all data under a new `id` and name "<name> copy" (numbered if taken) and drops the sync link and encryption key ([SYNC](sync.md)). Delete removes the local copy; deleting the server copy is a separate, explicit choice. | duplicate synced file → copy has no sync link |
| FILE-4 | Local-first: data is stored on the device and every operation except sync ([SYNC](sync.md)) and bank feeds ([IMP-12](import.md)) works with no network. Writes are durable before the operation reports success. | airplane mode → add transaction, reopen app → present |
| FILE-5 | Money is a signed integer count of the currency's minor unit; inflow > 0, outflow < 0. Allowed range \|x\| ≤ 2^51 − 1; outside → `amount_out_of_range`. All sums, differences and balances are exact integer arithmetic; no floating point is stored. | 0.1 + 0.2 entered as two amounts in USD → 10 + 20 = 30 |
| FILE-6 | `currency_code`: ISO 4217, chosen at create (default from device locale, else `USD`); `minor_digits` comes from ISO 4217 (0–3, e.g. USD 2, JPY 0, KWD 3). Changing the code is allowed only to a currency with the same `minor_digits`, else `currency_change_blocked`; stored integers never change. Display preferences (no effect on stored data): number format `1,000.00`\|`1.000,00`\|`1 000,00`\|`1'000.00`\|`1,00,000.00`, symbol before/after, hide fraction (display only, values unchanged), first day of week, date display format. | file in USD → change to EUR ok; to JPY → `currency_change_blocked`; hide fraction on, 1250 stored → shown "13"; stored stays 1250 |
| FILE-7 | One currency per file: every account and amount is in `currency_code`; no exchange rates, conversion or per-account currency. A person with money in two currencies keeps two files. | manual: account form has no currency field |
| FILE-8 | Text → integer: the decimal value is scaled by 10^`minor_digits` with exact decimal arithmetic and rounded once, half away from zero. This is the only place a fraction of a minor unit can arise outside the averages and percentages defined in [BUD](budget.md) and [RULE](rules.md), which round the same way. | USD "12.345" → 1235; "-12.345" → -1235; JPY "99.5" → 100 |
| FILE-9 | Amount text parsing: optional sign or parentheses (negative); optional currency symbol and spaces ignored; the last `.` or `,` is the decimal mark when followed by 1–2 digits, or by more than 3 digits; followed by exactly 3 digits it is a thousands separator. Anything else → `invalid_amount`. | "1,234" → 1234.00; "1,23" → 1.23; "(5.00)" → -5.00 |
| FILE-10 | Export all my data (`@specfork/privacy:PRV-1`), `format` `@specfork/budget`, `version` 1, covers one budget file and works offline; the manifest adds `name` and `currency_code`. One JSON Lines file per entity (`accounts`, `transactions`, `payees`, `category_groups`, `categories`, `budget_cells`, `budget_months`, `rules`, `schedules`, `reports`, `notes`, `tags`, `preferences`) holds every live row with all fields of this spec, plus `transactions.csv` (FILE-11). Deleted rows (sync tombstones, [SYNC-4](sync.md)), sync state, encryption keys and bank-feed credentials are not included. | 3 accounts, 40 transactions → manifest counts 3 and 40; zip has no key material |
| FILE-11 | `transactions.csv`: header `date,account,payee,category_group,category,amount,notes,cleared,reconciled,split_of`; amount as a decimal string with `minor_digits` places and `.` mark; split children carry the parent `id` in `split_of`, parents have empty category. Cells starting with `=`, `+`, `-`, `@`, tab or CR are prefixed with `'`. | payee "=SUM(A1)" → cell `'=SUM(A1)` |
| FILE-12 | Import (`@specfork/privacy:PRV-5`) of an export ZIP creates a new budget file (new `id`, name made unique per FILE-1) whose data equals the exported data; another `format` or an incompatible `version` → `unsupported_export`. | export, import → same balances, budget and rules |
| FILE-13 | Inventory (`@specfork/privacy:PRV-9`): on the device, the budget files of FILE-1, kept until the person deletes them; on the sync server, account data, snapshots and change messages (encrypted under [SYNC-10](sync.md)), kept until deleted or reset ([SYNC-8](sync.md), [SYNC-9](sync.md)). Third party: a bank-feed adapter per linked account, which receives only `external_id` and `since` and is called only while the account is linked ([IMP-12](import.md), `@specfork/privacy:PRV-10`). | unlink feed → no further adapter calls |

## Out of scope

- Importing whole budgets from other budgeting apps; automatic local backups and undo history.
- Multi-currency within one file, exchange rates, investment prices.

## Decisions

- One currency per file instead of per-account currencies: the source has only a file-level currency setting, and envelope arithmetic across currencies needs rates the spec would have to invent.
- Rounding is half away from zero everywhere; the source rounded halves toward positive infinity, which treats outflows and inflows differently.
- The export is `@specfork/privacy` JSON Lines plus CSV rather than the source's raw database file, so it is readable without the app.
- Exports are written on the device, not served by a server: `@specfork/privacy:PRV-11` local mode applies; the optional sync server uses server mode.
