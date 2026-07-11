# Salary References and Counterfactual-Cost Methodology

This document accompanies `development-log.csv` and specifies the salary rates,
sources, exchange rate, and profile-assignment rule used to compute the
**counterfactual** (without-AI) cost estimates. It is part of the replication
package and contains no author-, institution-, or project-identifying data.

All monetary values in `development-log.csv` are expressed in **US dollars
(USD)**. Brazilian real (BRL) equivalents are shown here in parentheses only to
document how the rates were derived; no BRL column is stored in the CSV.

## 1. Hourly rates (USD)

The counterfactual cost of each activity is
`counterfactual_hours × hourly_rate_usd`, where the rate is fixed by the profile
the **activity** requires:

| Profile (`counterfactual_profile`) | Hourly rate (`hourly_rate_usd`) | BRL equivalent |
| ---------------------------------- | ------------------------------- | -------------- |
| Junior                             | US$ 3.86 / h                    | (R$ 22 / h)    |
| Mid-level                          | US$ 6.67 / h                    | (R$ 38 / h)    |
| Senior                             | US$ 10.88 / h                   | (R$ 62 / h)    |

These three rates are the only rates used anywhere in `development-log.csv`.

**Derivation.** Each rate is a regional gross **monthly** salary divided by 160
working hours per month, converted to USD at the fixed rate of R$ 5.70 / USD
(see §3):

| Profile   | Gross monthly (regional) | ÷ 160 h = BRL/h | ÷ 5.70 = USD/h |
| --------- | ------------------------ | --------------- | -------------- |
| Junior    | R$ 3,520 / mo            | R$ 22 / h       | US$ 3.86 / h   |
| Mid-level | R$ 6,080 / mo            | R$ 38 / h       | US$ 6.67 / h   |
| Senior    | R$ 9,920 / mo            | R$ 62 / h       | US$ 10.88 / h  |

Rates are rounded to two decimals for display; because `counterfactual_hours`
values are integers, every non-empty `counterfactual_cost_usd` cell equals
`counterfactual_hours × hourly_rate_usd` **exactly** (verified programmatically).

## 2. Sources

Regional market rates in Brazil, derived from public software-engineering salary
surveys and job-market aggregators for the region where the project was
developed. **No city or state is named**, to preserve double-blind review; the
rates reflect the regional labour market in which the work actually took place,
not a national average or an international (e.g. SP/RJ or offshore) reference.

## 3. Exchange rate

A **fixed** rate of **R$ 5.70 / USD** is used for every BRL↔USD conversion in
the study (both the derivation of the hourly rates above and any AI-cost figure
originally recorded in BRL). Fixing the rate removes date-of-conversion variance
and makes the figures reproducible.

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

## 5. AI-tooling cost

The real cost of AI assistance is taken from **invoices and flat subscriptions**,
not from token-metered billing:

- **GitHub Copilot Business** — flat subscription (not billed per token); its
  cost is the pro-rated share of the subscription over the project window.
- **Abacus AI (ChatLLM)** — platform-credit consumption exported from the
  provider's billing log.
- **OpenAI API** — metered embeddings/completions, confirmed from the platform
  dashboard.

Invoice/subscription basis, these three total **≈ US$ 69** for the whole project.

> **Note on the per-activity `ai_cost_usd` column.** That column is a *token-based
> estimate* provided for per-activity granularity and sums to **US$ 33.66**. It is
> deliberately **not** equal to the billed AI cost above: Copilot is a flat
> subscription, so its real cost cannot be reconstructed from tokens. Use the
> invoice/subscription figure (≈ US$ 69) for any economic-ratio comparison; the
> token column is illustrative only.

## 6. Notes on the join and empty cells

`development-log.csv` has **one row per activity** at the finest granularity
available in the source. The source records three separate layers per phase
(AI consumption, real human effort, counterfactual estimate), and these layers
were authored with **different activity decompositions**. Rows join the three
layers by activity where they correspond 1:1; where a layer has no matching
activity, the corresponding cells are left **empty** (no value is invented).
The activities affected by such gaps are listed in the delivery report that
accompanies this package. Column totals are unaffected: every source value
appears exactly once, so the consolidated sums are preserved
(counterfactual = 329 h / **US$ 3,005.55** at hourly_rate_usd × counterfactual_hours;
real human effort = 35.2 h; tokens = 952,000 in / 837,000 out; per-activity
token-based AI-cost estimate = US$ 33.66; invoice/subscription AI cost ≈ US$ 69).
