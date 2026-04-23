---
name: sample-size
description: Describe and assess the sample size of a dataset — not just row count, but effective sample size per question the user wants to answer. Flags underpowered segments, imbalanced classes, small-n group cells, and gives a concrete "you can / cannot reliably claim X from this data" verdict.
---

# Sample Size

Row count is not sample size. This skill characterises the *effective* sample the user has for the *specific* analysis they want to run, and tells them whether that's enough.

## Inputs

- Path to a dataset (CSV / Parquet / Excel / DuckDB table).
- Optional: the analytical question (e.g. "compare conversion rate between A and B", "estimate mean spend per country", "train a classifier on 12 features"). If not provided, produce a general description only.
- Optional: target column, predictor columns, grouping columns, effect size / precision the user cares about.

## Recommended CLI tooling

- `duckdb` — counts, group-by cardinality, null rates.
- `uv run --with statsmodels python -c '...'` — power calculations (`statsmodels.stats.power`), sample-size-for-proportions, sample-size-for-regression.

## Procedure

### Step 1 — Raw size description

Report:
- **Total rows** and **total columns**.
- **Non-null rows** per column (effective n per variable).
- **Unique rows** (if duplicates matter — e.g. a "customers" table with repeated IDs isn't really that many customers).
- **Time span** (if a date column exists): first timestamp, last timestamp, duration, record density (rows/day).
- **Memory footprint** — helpful for deciding between in-memory vs. chunked processing.

### Step 2 — Unit of analysis

Ask (or infer): what is one row? Customer? Transaction? Country-year? Sensor reading?

The honest sample size for most claims is the **number of independent units**, not the number of rows. If the dataset has 1M transactions from 50 customers and the question is about customer behaviour, n = 50, not 1,000,000. Surface this distinction — it's the single biggest source of misstated power.

### Step 3 — Per-group and per-class cells

For each grouping column or categorical target the user mentions:
- **Group sizes**: the count of rows in each level.
- **Smallest cell**: the minimum count across groups. This is the binding constraint on any comparison.
- **Imbalance ratio**: largest / smallest. Flag if > 10:1 — statistical tests and ML models both degrade.
- For cross-tabs (two categorical vars): the smallest cell in the contingency table. Chi-square starts being unreliable when any expected cell is < 5.

### Step 4 — Effective sample size given missingness

Compute, per planned analysis column-set, the count of rows with **all** required columns non-null. This is frequently much smaller than the total row count — flag it loudly if drop rate > 20%.

### Step 5 — Adequacy rules of thumb

Apply standard rules of thumb, tuned to the user's stated question:

| Question type | Rough minimum n | Notes |
|---|---|---|
| Single-group mean with SE ≈ SD/10 | ~100 | For ±10%-of-SD precision. |
| Proportion estimate (±5 pp, 95% CI) | ~385 | Classic survey number. Scales down for wider CIs. |
| Two-group comparison, medium effect (Cohen's d = 0.5) | ~64 per group | 80% power, two-sided α = 0.05. |
| Two-group comparison, small effect (d = 0.2) | ~394 per group | 80% power. |
| Chi-square 2×2 | Min expected cell ≥ 5 | Use Fisher's exact below that. |
| Linear regression | ≥ 10–20 rows per predictor | Harrell's rule. |
| Logistic regression | ≥ 10–20 **events** per predictor (EPV) | Rare-event datasets bite here. |
| Any ML model | ≥ 10× predictors, more for non-linear / high-variance methods | Validate via CV, not rules of thumb alone. |
| Time series forecasting | ≥ 2 full seasonal cycles | Otherwise seasonality can't be separated from trend. |

For any quantitative question the user states with an effect size or precision target, run the matching power calculation in statsmodels and report the required n alongside the observed n.

### Step 6 — Verdict per question

For each stated analysis:
- **Green**: enough data to reliably answer at the stated precision.
- **Yellow**: marginal — results will have wide CIs, or only a large effect will be detectable. State what effect size is actually detectable at 80% power given the observed n.
- **Red**: underpowered — name the smallest cell or missingness that kills it, and what n would be needed.

### Step 7 — Report

Write `outputs/sample-size/report.md`:

- **Size snapshot** (Step 1).
- **Unit of analysis** and the honest n (Step 2).
- **Group / cell table** with the smallest-cell flag (Step 3).
- **Effective n per analysis** after missingness (Step 4).
- **Adequacy verdicts** per question (Step 6), each with a one-line "so what" — e.g. "Can detect a difference of 0.3 SD or larger with 80% power; smaller effects will be inconclusive."

Keep it short. This skill's job is to prevent overclaiming, not to teach statistics.

## Notes

- For clustered / nested data (repeated measures, pupils within schools, transactions within customers), the **design effect** reduces effective n further. If the user's question involves such structure, note it and suggest a mixed-effects or cluster-robust approach rather than treating each row as independent.
- For heavily imbalanced classification, report the event count and **events per predictor** — that's the real constraint, not total n.
- Pairs with `hypothesis-testing` (which consumes the verdict to decide whether the test is worth running) and `multivariate-analysis` (where the 10-EPV / 10-rows-per-predictor rule bites hardest).
