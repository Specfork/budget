# Accounts

**Status:** Draft · **Prefix:** ACC

## Purpose

Where money sits: on-budget accounts feed the envelope budget, off-budget accounts are only
tracked. Balances, opening balance, closing, deleting and reconciling against a statement.

## Requirements

| ID | Requirement | Verified by |
|---|---|---|
| ACC-1 | Account: `id`, `name` (trimmed, 1–100 chars, unique case-insensitively among open accounts, else `invalid_name`), `on_budget` (bool, set at create, immutable), `closed` (default false), `sort_order`, `note?`, `last_reconciled_at?`, `feed_link?` ([IMP-12](import.md)). New accounts sort last among accounts with the same `on_budget`. | create two "Card" → second `invalid_name` |
| ACC-2 | Creating an account also creates its transfer payee ([CAT-2](categories.md)). | create "Savings" → payee list has transfer payee for "Savings" |
| ACC-3 | Opening balance `b` at create: `b ≠ 0` → one transaction dated today, amount `b`, payee `starting_balance` (system payee, created once), `cleared` = true, `is_opening_balance` = true; category `starting_balances` (recreated in the income group if missing) when on-budget, none when off-budget. `b = 0` → no transaction. | on-budget, b = 50000 → to-budget this month +50000 |
| ACC-4 | `balance` = sum of `amount` over the account's live transactions that are not split parents; `cleared_balance` = same over `cleared` = true; `uncleared_balance` = `balance` − `cleared_balance`. | +1000 cleared, −300 uncleared → 700 / 1000 / −300 |
| ACC-5 | Budget totals: on-budget balance = Σ `balance` of open and closed on-budget accounts; off-budget likewise; net worth = both. | on 700, off 5000 → net worth 5700 |
| ACC-6 | Close: `balance` = 0 → `closed` = true. `balance ≠ 0` needs `transfer_to` (another open account, not itself, else `invalid_transfer_account`; missing → `balance_not_zero`); the app adds a transfer of −`balance` dated today with note "account closed" to it, then closes. On-budget → off-budget needs `category` for the outflow, else `category_required`. | close 700 into savings → savings +700, closed account 0 |
| ACC-7 | Closed accounts: hidden from account pickers and the sidebar's open list; new transactions in them → `account_closed`; existing ones stay editable and count in reports. Reopen sets `closed` = false. | add to closed account → `account_closed` |
| ACC-8 | Delete: removes the account, its transactions and its transfer payee. Transfer counterparts in other accounts stay, with `payee` and `transfer_id` cleared ([TXN-12](transactions.md)). Confirmation required. | delete checking with 1 transfer to savings → savings row kept, no payee |
| ACC-9 | Reconcile start: the person enters `statement_balance` (default `cleared_balance`); `difference` = `statement_balance` − `cleared_balance`, recomputed as rows are cleared or uncleared. Reconcile adjust: with `difference ≠ 0` the person may add an adjustment transaction: amount `difference`, dated today, `cleared` = true, note "reconciliation adjustment", no category unless rules set one ([RULE](rules.md)). | statement 1200, cleared 1000 → difference 200; difference 200 → adjust → new row +200, difference 0 |
| ACC-10 | Reconcile finish: allowed only when `difference` = 0, else `not_balanced`; sets `reconciled` = true on every cleared, unreconciled transaction of the account and `last_reconciled_at` = now. | difference 5 → finish → `not_balanced`, nothing locked |
| ACC-11 | Reconciled lock: changing `amount`, `date`, `account` or `payee` of a reconciled transaction, or of a transfer whose counterpart is reconciled, needs `unlock` = true, else `reconciled_locked`. Clearing `cleared` on a reconciled row sets `reconciled` = false (needs `unlock`). Moving it to another account sets `reconciled` = false. Bulk cleared toggles skip reconciled rows. | edit reconciled amount without unlock → `reconciled_locked` |

## Out of scope

- Account types (checking, credit, loan…), interest, credit limits, investment holdings.
- Account groups or folders.

## Decisions

- Closing an empty account closes it; the source deleted it silently.
- `on_budget` is fixed at create: switching would rewrite past budget months.
- The reconciled lock is enforced by the data layer, not only by the editor as in the source.
- New rows in closed accounts are refused so closed balances stay at zero.
