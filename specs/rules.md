# Rules

**Status:** Draft · **Prefix:** RULE

## Purpose

If-then rules that clean up payees, set categories, split and annotate transactions automatically.

## Requirements

| ID | Requirement | Verified by |
|---|---|---|
| RULE-1 | Rule: `id`, `stage`: `pre`\|`default`\|`post` (default `default`), `match`: `all`\|`any` (default `all`), `conditions` (≥ 1), `actions` (≥ 1), `created_at`. Empty conditions or actions, or an unknown stage → `invalid_rule`. | rule with no conditions → `invalid_rule` |
| RULE-2 | Condition fields and operators: text `imported_payee`, `notes`: `is`, `is_not`, `contains`, `not_contains`, `matches` (regex), `one_of`, `not_one_of` (not for notes), `has_tags`; reference `payee`, `category`, `category_group`, `account`: `is`, `is_not`, `one_of`, `not_one_of`, plus `account` `on_budget`\|`off_budget`; `date`: `is`, `is_approx`, `before`, `on_or_before`, `after`, `on_or_after`; `amount`, `amount_in`, `amount_out`: `is`, `is_approx`, `between`, `gt`, `gte`, `lt`, `lte`; `cleared`, `reconciled`, `is_transfer`, `is_split_parent`: `is`. Other pairs → `invalid_rule`. | `notes one_of` → `invalid_rule` |
| RULE-3 | Text comparisons ignore case. `contains`, `matches`, `has_tags` need a non-empty value. An invalid regex → `invalid_rule` on save. An empty or missing text field compares as "". A missing reference matches `is` nothing. | rule notes contains "COFFEE", notes "coffee shop" → match |
| RULE-4 | `amount is_approx v` matches \|amount − v\| ≤ round_half_away(\|v\| × 0.075). `date is_approx d` matches within ±2 days, inclusive. `between {a, b}` is inclusive and order-free. `date is` accepts `YYYY-MM-DD`, `YYYY-MM` (whole month) or `YYYY` (whole year). | approx −10000 → −10750..−9250 match; −10751 no |
| RULE-5 | `amount_in` considers only amounts > 0; `amount_out` only amounts < 0 and compares their absolute value. | `amount_out gt 5000`, amount −6000 → match; +6000 → no |
| RULE-6 | Order: all `pre` rules, then `default`, then `post`. Within a stage, ascending score, ties by `created_at`. Score = Σ per condition (`is`/`is_not` 10, `one_of`/`not_one_of` 9, `is_approx`/`between` 5, `gt`/`gte`/`lt`/`lte`/date bounds 1, others 0), doubled when every condition is `is`, `is_not`, `is_approx`, `one_of` or `not_one_of`. Actions apply in sequence, so a later (more specific) rule overwrites an earlier one; each rule sees the result of earlier rules. | "payee is A" (score 20) and "notes contains x" (0) both set category → payee rule wins |
| RULE-7 | Actions: `set` field (`payee`, `payee_name` creating the payee per [CAT-1](categories.md), `category`, `account`, `date`, `amount`, `notes`, `cleared`); `prepend_notes`, `append_notes` (no separator added); `link_schedule`; `split` (RULE-8); `delete_transaction`. | append " #work" to "lunch" → "lunch #work" |
| RULE-8 | Split action: a list of parts, each `fixed` (amount), `percent` (0–100) or `remainder`, with optional set actions per part; skipped on split children. Fixed parts first; `rest` = total − Σ fixed; each percent part = round_half_away(`rest` × p / 100); `rest2` = `rest` − Σ percent parts; `remainder` parts get `trunc(rest2 / k)` each and the last gets what is left. | total −10000: fixed −2000, 50 %, 2 remainders → −2000, −4000, −2000, −2000 |
| RULE-9 | When rules run: on every imported or fed transaction before duplicate matching ([IMP-9](import.md)); on a new hand-entered transaction, filling only fields the person left empty (notes actions still apply); on schedule posting; on demand for selected existing transactions. Saving or editing a rule never changes existing transactions by itself; the editor shows matching transactions and can apply the rule's actions to chosen ones. A transaction linked to a schedule ([SCH-6](schedules.md)) keeps getting that schedule's actions even when its amount or date later drifts outside the schedule's match. | save rule → past rows unchanged until "apply"; schedule amount drifts 10 % → still categorised |
| RULE-10 | Payee clean-up rule: stage `pre`, condition `imported_payee one_of [names]`, action `set payee T`; created or extended by payee merge with `remember` ([CAT-3](categories.md)). One such rule per target payee. | merge twice into T → one rule, names unioned |
| RULE-11 | Category learning (setting `learn_categories`, default on; payee flag [CAT-1](categories.md)): after a transaction with a payee and category is saved, take the payee's 5 latest non-parent transactions in open accounts within 180 days before to 180 days after the saved one; if one category appears ≥ 3 times, set it on the `default` rule "payee is P → set category" (create it if missing; leave alone if a `one_of` rule already covers P). | 3 of last 5 "grocer" rows → food → rule "payee is grocer → food" exists |

## Out of scope

- Templated or formula-based action values.
- Rules on budget cells or accounts.

## Decisions

- Invalid regexes are refused on save; the source accepted them and never matched.
- Rules do not rerun on save, as in the source; applying them to history is explicit.
