# Import

**Status:** Draft · **Prefix:** IMP

## Purpose

Bring bank data in from statement files or a bank feed without creating duplicates.

## Requirements

| ID | Requirement | Verified by |
|---|---|---|
| IMP-1 | File import into one account; type by extension: `.csv`, `.tsv` → CSV; `.ofx`, `.qfx` → OFX; `.qif` → QIF; else `unsupported_file`. Max 20 MB and 50,000 rows, else `file_too_large`. | `.pdf` → `unsupported_file` |
| IMP-2 | CSV options: `delimiter` `,` (default; tab for `.tsv`)\|`;`\|`\|`\|tab\|`~`; `has_header` (default true); `skip_start`, `skip_end` lines (0–1000; their sum ≥ line count → `nothing_to_import`); `encoding` auto (BOM → UTF-16LE/BE, else UTF-8) or `utf-8`, `utf-16le`, `utf-16be`, `windows-1252`, `windows-1250`, `iso-8859-2`. Quoted fields with `"`; blank lines skipped; cells trimmed. | `;` file with header → columns split |
| IMP-3 | CSV mapping: columns for `date` (required), `payee`, `notes`, `category` (exact category name, else empty) and amount by one of: `amount`; `inflow` + `outflow` (each read as absolute value; outflow wins when both set); `amount` + `direction` column with an `out_value` text meaning outflow. Options `flip_sign` (default false), `multiplier` (decimal > 0, default 1, applied before FILE-8 rounding). Missing date or amount mapping → `mapping_incomplete`. | inflow "", outflow "12.50" → −1250 |
| IMP-4 | CSV `date_format`: `yyyy mm dd`\|`yy mm dd`\|`mm dd yyyy`\|`mm dd yy`\|`dd mm yyyy`\|`dd mm yy`; any non-digit separator or none (fixed positions); English month names and 3-letter abbreviations accepted; 2-digit years are 20yy. Default: first format that parses row 1. A row whose date or amount does not parse aborts the import with `invalid_row` and the row number. | `dd mm yy`, "03/04/26" → 2026-04-03 |
| IMP-5 | OFX/QFX (SGML or XML; bank, card and investment cash statements): date = first 8 digits of the posted date; amount from the transaction amount ([FILE-9](files.md)); `imported_id` = the institution's transaction id; payee = name, notes = memo with markup removed. Empty name with `memo_as_payee` (default true) → memo becomes payee and notes are empty; `swap_payee_memo` (default false) swaps them. | empty name, memo "COFFEE 12" → payee "Coffee 12" |
| IMP-6 | QIF: must start with a `!Type:` line, else `invalid_file`; records end with `^`; codes date, amount, payee, memo, cleared flag, category (`group:category` → category), number; split lines (category, memo, amount) become split children ([TXN-6](transactions.md)). Dates use the CSV date formats (IMP-4). Records without date or amount are skipped and counted. | QIF record with 2 split lines → 1 parent, 2 children |
| IMP-7 | Optional `start_date`: rows dated earlier are dropped before matching. | start 2026-03-01 → February rows ignored |
| IMP-8 | Per-file options: `cleared` for new rows (default true), `import_notes` (default true), `payee_case`: `as_is`\|`title_case` (default `title_case`). The last options used per account (mapping, delimiter, encoding, header, skips, sign, format) are remembered and offered next time. | reopen import for same account → previous mapping preselected |
| IMP-9 | Pipeline for every incoming row: trim payee text → `imported_payee`; apply `payee_case`; resolve payee per [CAT-1](categories.md); run rules ([RULE-9](rules.md)); match (IMP-10); insert or merge; link schedules ([SCH-6](schedules.md)). The whole import is one atomic change. Preview runs the same pipeline without writing and returns each row as `new`, `matched` (with the existing row) or `skipped`. | preview → nothing saved, counts shown |
| IMP-10 | Matching in the target account: (1) same `imported_id` (deleted rows included unless `reimport_deleted` = true); (2) else candidates with equal `amount` and date within ±7 days, not already matched in this import, and for files not carrying a different `imported_id`; prefer same payee, then nearest date, then candidates without `imported_id`. | existing −4500 on 3 Mar, import −4500 on 5 Mar → matched |
| IMP-11 | Merge on match: reconciled rows are left unchanged; else set `imported_id` and `imported_payee`, fill `payee`, `category`, `notes` only if empty, `cleared` = existing OR incoming; `date` changes only with `update_dates` (default false). | existing uncleared, import cleared → cleared |
| IMP-12 | Bank feed contract (provider-neutral adapter): `list_accounts()` → `[{external_id, name, mask?, currency, balance}]`; `fetch(external_id, since)` → `{balance, transactions: [{external_id?, booked: bool, date, amount (minor units), payee_name, notes?, raw}]}`; errors `auth_required`, `rate_limited`, `account_missing`, `unavailable`. A feed currency ≠ the file currency → `currency_mismatch`, account not linked. Link: an account gets `feed_link` {`adapter`, `external_id`, `last_sync_at`, `status`: `ok`\|`auth_required`\|`rate_limited`\|`account_missing`\|`unavailable`}; one external account links to at most one account. Unlink clears `feed_link` and keeps all transactions. | adapter stub with 2 accounts → link dialog lists 2; unlink → rows kept, no sync |
| IMP-13 | Sync of a linked account: `since` = `last_sync_at` − 7 days, or on first sync max(today − 90 days, custom start); rows go through IMP-9 with `cleared` = `booked`; `pending_import` (default true) false → non-booked rows skipped. First sync adds an opening balance (ACC-3 rules) = feed `balance` − Σ fetched amounts, dated at the oldest fetched row. Pending rows: a pending row is replaced when the feed returns a booked row with its `external_id`; a pending, unreconciled row the person has not edited that is absent from a later fetch covering its date is deleted. | balance 1000, fetched −200, +50 → opening 1150; pending −30 gone from feed → removed |

## Out of scope

- Named bank-sync vendors and their credential flows; the adapter owns credentials and they never enter the budget file.
- CAMT, MT940 and PDF statements; investment trades from OFX.

## Decisions

- QIF splits import as splits; the source dropped them.
- Stale pending rows are removed; the source left them.
- Feed start window 90 days (source: 89).
