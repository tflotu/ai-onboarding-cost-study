# Replication Package

Supporting data for the experience report *"Building AI-Intensive Software with
AI: An Experience Report on Cost and Process."*

This package contains the records behind the cost analysis reported in the
paper. It contains no author-, institution-, or project-identifying
information.

## Contents

```
data/
  development-log.csv     One row per recorded activity: tokens, AI cost,
                          real human effort, and the counterfactual estimate.
  salary-references.md    Hourly rates, sources, exchange rate, and the
                          profile-assignment rule used for the counterfactual.
prompts/
  prompt-catalogue.md     The ten versioned application prompts (PROMPT-001
                          to PROMPT-010) embedded in the system.
```

## Verifying the paper's figures

All monetary values are in USD. Most headline figures are the sum of a column
in `development-log.csv`:

| Paper figure | Source | Value |
| --- | --- | --- |
| Input tokens | `input_tokens` | 952,000 |
| Output tokens | `output_tokens` | 837,000 |
| Actual human effort | `human_hours_ai` + `human_hours_review` | 35.2 h |
| Counterfactual hours | `counterfactual_hours` | 329 h |
| Counterfactual cost | `counterfactual_cost_usd` | US$ 3,005.55 |
| Real AI spend | invoices/subscription (see `salary-references.md` §5) | ~US$ 69 |

Each non-empty `counterfactual_cost_usd` equals
`counterfactual_hours × hourly_rate_usd` exactly.

**Important:** the `ai_cost_usd` column is a per-activity *token-based estimate*
and sums to US$ 33.66. This is **not** the figure the paper uses for the cost
ratio. Because the coding agent runs under a flat-rate subscription, real AI
cost comes from invoices (~US$ 69), not token counts. The paper treats the
token estimate as a corrected measurement error; `salary-references.md` §5
explains the two bases. Use ~US$ 69 for the 9.9x ratio.

## Known limitations of these records

We state these plainly, because they bound what the data can support.

**Tokens are estimated, not metered.** Counts were approximated from
conversation volume rather than read from provider telemetry. They are
order-of-magnitude figures.

**Human effort is self-reported.** The `human_hours_*` columns were logged by
the team, not instrumented.

**The counterfactual was never performed.** The `counterfactual_*` columns
estimate what a professional team would have needed *without* AI assistance.
They are retrospective judgments by student developers, not a measured control
condition, and were not calibrated against a professional baseline or
historical project data. They are subject to hindsight bias.

**Some cells are empty.** The source recorded three layers (AI consumption,
real human effort, counterfactual estimate) using *different activity
decompositions*. Rows join the layers where activities correspond one-to-one;
where a layer has no matching activity, cells are left empty rather than
imputed. No value was invented. Column totals are unaffected: every source
value appears exactly once.

## License

[TODO: choose a license, e.g. CC BY 4.0 for data.]
