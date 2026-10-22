# Transactions

**Status:** Draft · **Prefix:** TXN

## Purpose

Money moving in or out of an account, split across categories, or moving between two accounts.

## Requirements

| ID | Requirement | Verified by |
|---|---|---|
| TXN-1 | Transaction: `id`, `account` (required), `date` (`YYYY-MM-DD`, 1900-01-01–2099-12-31, else `invalid_date`), `amount` ([FILE-5](files.md), required), `payee?`, `category?`, `notes?` (≤ 4,000 chars), `cleared`, `reconciled` (default false), `parent_id?`, `transfer_id?`, `schedule_id?`, `imported_id?`, `imported_payee?`, `is_opening_balance` (default false), `split_error?`, `sort_order`, `created_at`. | date 2100-01-01 → `invalid_date` |
| TXN-2 | `cleared` default: false for transactions entered by hand or posted by a schedule; imports use the import's setting ([IMP-8](import.md)). | add by hand → cleared false |
| TXN-3 | `category` is always empty on transactions in off-budget accounts, on split parents and on transfers between two accounts with the same `on_budget`; a value sent there is dropped. Only categories of the expense or income kind may be set ([CAT](categories.md)). | off-budget row with category "food" → saved without category |
| TXN-4 | List order within an account: `date` descending, then opening-balance rows last within their date, then `sort_order` descending (newer entries first), then `id`. Split children follow their parent in their own `sort_order`. The person may move a row only among rows with the same account and date. | two rows same date → later-entered listed first |
| TXN-5 | Delete is soft: the row is marked deleted, hidden everywhere and excluded from every sum; it syncs as a field change ([SYNC-4](sync.md)). Deleted rows are purged only by sync reset ([SYNC-9](sync.md)). | delete −50 → account balance +50 |
| TXN-6 | Split: a parent with ≥ 1 children (`parent_id` = parent). Children have their own `amount`, `payee`, `category`, `notes`, and take `account`, `date`, `cleared`, `reconciled` from the parent (changing the parent's changes theirs). The parent's own category is empty; balances and budgets count children only, never the parent ([ACC-4](accounts.md), [BUD-3](budget.md)). | parent −100: food −60, home −40 → food activity −60 |
| TXN-7 | Split mismatch: `split_error` = parent `amount` − Σ children `amount`; saved even when ≠ 0 and flagged on the parent until fixed. Converting a row into a split adds one child with amount = parent amount unless that is 0. | parent −100, children −60, −30 → `split_error` −10 |
| TXN-8 | Distribute: spreads `split_error` over the children whose amount is 0: each gets `trunc(split_error / n)`, then the leftover \|r\| minor units go one each, sign of `split_error`, to the first such children in list order. | error −100, 3 zero children → −34, −33, −33 |
| TXN-9 | Deleting a parent deletes its children. Deleting a child that leaves no children turns the parent into a normal transaction keeping its `amount`. | delete only child → parent normal, no `split_error` |
| TXN-10 | Transfer: setting `payee` to the transfer payee of account B on a row in account A creates a counterpart in B: `amount` negated, same `date` and `notes`, `payee` = A's transfer payee, `cleared` = false, both rows linked by `transfer_id`. Transfer to the same account → `invalid_transfer`. A split parent cannot be a transfer; its children can. | A −500 to B → B +500, linked |
| TXN-11 | Editing either side of a transfer updates the other: `amount` (negated), `date`, `notes`, and the target account when the payee changes to another transfer payee. Category follows TXN-3: kept only when exactly one side is on-budget, and then only on the on-budget side. | change A side to −600 → B side +600 |
| TXN-12 | Removing the transfer payee from one side, or deleting one side, deletes the counterpart; if the counterpart is a split child it is kept with `payee` and `transfer_id` cleared. | delete A side → B side gone |
| TXN-13 | Bulk edit: set `date`, `account`, `payee`, `category`, `notes` (replace, prepend or append) or `cleared` on selected rows in one atomic change; each row still obeys TXN-3 and [ACC-11](accounts.md). Filter transactions by any condition of [RULE-2](rules.md) combined with all/any, plus free-text search over payee, category, notes and amount text; results show count and Σ `amount`. | select 3 → set category food → 3 changed; filter payee is "grocer", Jan → 4 rows, Σ −23000 |

## Out of scope

- Attachments and receipts, investment lots, pending-hold amounts.

## Decisions

- Transfer dates always stay in step; the source kept them in step only behind a preference.
- Manual and scheduled rows start uncleared so reconciliation ([ACC-9](accounts.md)) reflects what the bank has not yet shown; the source defaulted every row to cleared.
- A mismatched split is saved and flagged, as in the source, so the person can finish it later.
