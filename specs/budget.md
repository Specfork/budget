# Budget

**Status:** Draft · **Prefix:** BUD

## Purpose

The monthly envelope budget: how much is assigned to each category, what was spent, what is
available, how leftovers and overspending move into the next month, and how much is left to assign.

## Requirements

| ID | Requirement | Verified by |
|---|---|---|
| BUD-1 | Budget months are `YYYY-MM` from (month of the earliest live transaction − 3 months) to (current month + 12); with no transactions, from current − 3. Months extend automatically as transactions or time move. Reading or writing outside → `month_out_of_range`. All formulas use whole calendar months of transaction `date`. | earliest 2026-05-10, today 2026-10-22 → 2026-02..2027-10 |
| BUD-2 | Budget cell per (expense category, month): `assigned` (integer minor units, any sign, default 0). Income categories have no cells. | new month → all `assigned` 0 |
| BUD-3 | `activity(c, m)` = Σ `amount` of live transactions with category `c`, dated in `m`, in on-budget accounts, not split parents (children count). Transfers between on-budget accounts have no category (TXN-3) and so never count. | on-budget −60 food + off-budget −40 food → activity −60 |
| BUD-4 | `available(c, m)` = `assigned(c, m)` + `activity(c, m)` + `carry(c, m)`, where `carry(c, m)` = `available(c, m−1)` if `rollover_overspending(c, m−1)`, else `max(available(c, m−1), 0)`; `carry` = 0 in the first month. A positive balance therefore always rolls forward. | assigned 500, activity −600 → available −100 |
| BUD-5 | `rollover_overspending(c, m)`: bool, default false. Setting it for month `m` sets it for `m` and every later month in range; clearing likewise. | set in March → true in March..last month |
| BUD-6 | `overspent(m)` = Σ over expense categories with `rollover_overspending(c, m−1)` = false of `min(available(c, m−1), 0)` (≤ 0). The negative part not rolled forward in BUD-4 is charged here, once, to next month's to-budget. | Jan food −100, not flagged → Feb `overspent` −100, Feb food carry 0 |
| BUD-7 | `income(m)` = Σ `activity` of all income categories in `m` (on-budget only, BUD-3). `to_budget(m)` = `income(m)` + `to_budget(m−1)` + `held(m−1)` + `overspent(m)` − Σ `assigned(c, m)` over all expense categories (hidden included) − `held(m)`. Month summary: `income`, `to_budget`, `held`, `overspent` (from BUD-6), `assigned_total` = Σ `assigned`, `activity_total` = Σ expense `activity`, `available_total` = Σ expense `available`. Group rows show Σ of their categories. | Jan: income 3000, food 500/−600, rent 1000/−1000, hold 200 → to_budget 1300; Feb: food 400/−100 → `overspent` −100, to_budget 1000, food available 300 (flagged in Jan: 200 and 1100); 2 categories assigned 500, 1000 → `assigned_total` 1500 |
| BUD-8 | The month before the first month has `to_budget` = 0, `held` = 0, every `available` = 0 and every flag false. | first month, income 3000, assigned 1000 → to_budget 2000 |
| BUD-9 | Hold: `held(m)` (default 0) sets money aside for next month. Hold `a` → `held(m)` += `clamp(a, −held(m), max(to_budget(m), 0))`; reset → `held(m)` = 0. `held(m)` re-enters as income-like money in `m+1` (BUD-7) and never resets on its own. | to_budget 300, hold 500 → held 300, to_budget 0 |
| BUD-10 | `to_budget(m)` may be negative ("over-assigned"); it is shown as such and carries into `m+1` unchanged. | Jan to_budget −50, Feb nothing else → Feb −50 |
| BUD-11 | Copy last month: for each visible expense category with a cell in `m−1`, `assigned(c, m)` = `assigned(c, m−1)`; per-category variant copies one. Hidden categories are skipped. Set to zero: every expense category's `assigned(c, m)` = 0, hidden included. Copy to later months: one category's `assigned(c, m)` copied to every later month through December of `m`'s year. | Dec food 400 → copy into Jan → 400; set zero on Jan → assigned_total 0 |
| BUD-12 | Average of N ∈ {3, 6, 12}: for each visible expense category, take up to N months immediately before `min(m, current month)`, stopping before the category's first month with a cell or activity; `k` = months taken; `assigned(c, m)` = round_half_away(−Σ `activity` / `k`); `k` = 0 → 0. | activity −100, −101, −102 over 3 months → 101 |
| BUD-13 | Move `a` > 0 from category X to Y in month `m`: `assigned(X, m)` −= `a`; `assigned(Y, m)` += `a`; no limit (X may go negative); `to_budget` unchanged. A line "moved a from X to Y" is appended to the month note. | X 300 → move 400 to Y → X −100, Y +400 |
| BUD-14 | From to-budget into Y: `a'` = `clamp(a, 0, max(to_budget(m), 0))`, `assigned(Y)` += `a'`. From X into to-budget: `assigned(X)` −= `a`. | to_budget 200, move 500 to food → food +200, to_budget 0 |
| BUD-15 | Cover overspending of Y (`available(Y, m)` < 0) from source S: `need` = −`available(Y, m)`; `got` = `min(need, available(S, m))` for a category, or `min(need, to_budget(m))` for to-budget; nothing happens if `got` ≤ 0; `assigned(S)` −= `got` (unless S is to-budget), `assigned(Y)` += `got`. Fix over-assigned (`to_budget(m)` < 0) from X: `r` = `min(−to_budget(m), max(available(X, m), 0))`; `assigned(X)` −= `r`. | Y −80, S available 50 → Y −30, S 0; to_budget −70, X available 50 → X assigned −50, to_budget −20 |

## Out of scope

- Goal and template engines (target balances, percentage-of-income, schedule-funded goals).
- The source's second budget type (income vs expense tracking without envelopes).
- Automatic holding of income for the next month.

## Decisions

- `income` sums every income category; the source used only the first income group (moot with one group, [CAT-5](categories.md)).
- Month range is computed from live transactions; the source also counted deleted ones.
- Averages round half away from zero ([FILE-8](files.md)).
