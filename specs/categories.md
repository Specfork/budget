# Categories

**Status:** Draft · **Prefix:** CAT

## Purpose

Who money goes to (payees) and what it is for (categories in groups), plus notes and tags.

## Requirements

| ID | Requirement | Verified by |
|---|---|---|
| CAT-1 | Payee: `id`, `name` (trimmed, 1–200 chars), `favorite` (default false), `learn_categories` (default true, [RULE-11](rules.md)), `transfer_account?`. Entering a payee by name reuses an existing payee whose name matches case-insensitively, else creates one. | type "acme" with "Acme" existing → same payee |
| CAT-2 | Each account has exactly one transfer payee (`transfer_account` set), shown with the account's current name; it cannot be renamed, deleted or merged (`system_payee`). | rename transfer payee → `system_payee` |
| CAT-3 | Merge payees P1…Pn into target T (not a transfer payee): transactions, rules and schedules pointing at any Pi point at T; Pi are deleted; with `remember` = true a pre-stage rule "imported payee one of [Pi names] → set payee T" is created or extended ([RULE-10](rules.md)). | merge "ACME INC" into "Acme" → rows show "Acme" |
| CAT-4 | Delete payee: its transactions keep their rows with `payee` empty; rules or schedules that name it → `payee_in_use` listing them. A payee with no transactions, rules or schedules is "unused" and the payee list offers to merge or delete it. | delete payee in a rule → `payee_in_use` |
| CAT-5 | Category group: `id`, `name` (trimmed, 1–100 chars, unique case-insensitively among groups), `is_income`, `hidden` (default false), `sort_order`. Exactly one income group exists; it cannot be deleted or hidden. Income groups sort after expense groups. | create second income group → `income_group_exists` |
| CAT-6 | Category: `id`, `name` (trimmed, 1–100 chars, unique case-insensitively within its group, else `name_taken`), `group`, `hidden` (default false), `sort_order`, `note?`. Its kind (income or expense) is its group's; moving it to a group of the other kind → `kind_mismatch`. New categories go first in their group unless placed. | "Food" and "food" in one group → `name_taken` |
| CAT-7 | Hidden categories and groups are left out of pickers, copy-last-month and averages ([BUD-11](budget.md)), but their money still counts in every total and report. Hiding a group hides its categories. | hide "gifts" with 3000 available → to-budget unchanged |
| CAT-8 | Delete category C with `move_to` D: all transactions, rules and schedules using C use D; for an expense C, each month's `assigned` of C is added to D's. `move_to` is required when C has transactions or any non-zero `assigned` (else `move_to_required`); D must be of the same kind (`kind_mismatch`) and not C. Delete group: deletes each of its categories as above into one `move_to` category outside the group of the same kind, then the group. | delete food (assigned Jan 300) into home (200) → home Jan 500; delete "fun" group into "misc" → all moved |
| CAT-9 | Notes: plain text ≤ 10,000 chars attached to an account, a category, a category in one month, or a budget month; empty text removes the note. | note on Jan budget → shown only for Jan |
| CAT-10 | Tags: a `#word` (no spaces or `#`) inside transaction notes; `##` writes a literal `#`. Tag: `name` (unique, case-sensitive), `color?`, `description?`, `hidden`. Renaming a tag rewrites it in every note; a name with space or `#` → `invalid_tag`. | rename #trip → #holiday → notes updated |

## Out of scope

- Payee locations, payee contact details, category icons.
- Nested categories deeper than group → category.

## Decisions

- One income group: the to-budget formula ([BUD-7](budget.md)) sums all income categories; the source summed only the first income group.
- Deleting a payee used by rules is refused rather than leaving rules pointing at nothing.
