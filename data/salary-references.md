KCA# Salary References and Counterfactual-Cost Methodology

This document accompanies `development-log.csv` and specifies the salary rates,
sources, exchange rate, and profile-assignment rule used to compute the
**counterfactual** (without-AI) cost estimates. It is part of the replication
package and contains no author-, institution-, or project-identifying data.

## 1. Hourly rates (U$)

The counterfactual cost of each activity is `counterfactual_hours × hourly_rate_US`,
where the rate is fixed by the profile the **activity** requires:

| Profile (`counterfactual_profile`) | Hourly rate (`hourly_rate_US`) |
| ---------------------------------- | ------------------------------- |
| Junior                             | u$ 3.86 / h                       |
| Mid-level                          | u$ 6.67 / h                       |
| Senior                             | u$ 10.88 / h                      |

These three rates are the only rates used anywhere in `development-log.csv`.
Every non-empty `counterfactual_cost_us` cell equals `hours × rate` exactly
(verified programmatically).

## 2. Sources

- **ABES (Brazilian Association of Software Companies) — 2025 salary survey.**
- **Glassdoor Brazil** — reported ranges for Junior / Mid-level / Senior
  software-engineering roles.

Rates are national-level reference figures; no city or regional qualifier is
used, to avoid geographic re-identification.

## 3. Exchange rate

A **fixed** rate of **R$ 5.70 / USD** is used for every USD→BRL conversion in
the study (e.g. AI cost in USD → BRL). Fixing the rate removes date-of-conversion
variance and makes the figures reproducible.

## 4. Profile-assignment rule

> The profile of an activity is assigned by the **expertise the activity itself
> demands**, **not** by the seniority of the team member who actually performed it.

Rationale: the counterfactual estimates the cost of a professional team building
the same system without AI assistance. The relevant question is therefore "what
level of professional would this task require?" — independent of who did it in
the actual (AI-assisted) run, where individual contributors frequently operated
above their nominal seniority precisely because AI removed the implementation
friction.

Consequences:
- The same activity type receives the same profile across phases.
- Architectural, security, and complex-backend work is rated **Senior**;
  feature/UI and integration work of moderate complexity is rated **Mid-level**;
  routine documentation and simple tasks are rated **Junior**.
- The self-reported human-effort columns (`human_hours_ai`, `human_hours_review`)
  are **not** rate-multiplied here and are independent of `counterfactual_profile`;
  they record the real, AI-assisted effort and are provided for transparency only.

## 5. Notes on the join and empty cells

`development-log.csv` has **one row per activity** at the finest granularity
available in the source. The source records three separate layers per phase
(AI consumption, real human effort, counterfactual estimate), and these layers
were authored with **different activity decompositions**. Rows join the three
layers by activity where they correspond 1:1; where a layer has no matching
activity, the corresponding cells are left **empty** (no value is invented).
The activities affected by such gaps are listed in the delivery report that
accompanies this package. Column totals are unaffected: every source value
appears exactly once, so the consolidated sums are preserved
(AI cost = US$ 33.66; counterfactual = 329 h / R$ 5,678; tokens = 952,000 in /
837,000 out; real human effort = 35.2 h).
