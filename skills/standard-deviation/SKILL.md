---
name: standard-deviation
description: Compute and interpret standard deviation (and related spread measures — variance, IQR, MAD, CV) for numeric columns in a dataset. Handles sample vs. population formulas, grouped/stratified computation, and flags columns where SD is misleading (heavy skew, outliers, near-constant values).
---

# Standard Deviation

Compute standard deviation for one or more numeric columns, plus the context needed to actually use the number: sample vs. population formula, comparison to related spread measures, and warnings when SD is the wrong summary.

## Inputs

- Path to a dataset (CSV / Parquet / Excel / DuckDB table).
- Optional: specific columns. Default: all numeric columns.
- Optional: grouping column — compute SD within each group.
- Optional: formula — `sample` (n-1 denominator, default) or `population` (n denominator). Default is sample, because almost all real data is a sample of something.

## Recommended CLI tooling

- `duckdb` — built-in `stddev_samp()`, `stddev_pop()`, `variance()`, `quantile_cont()`.
- `uv run --with pandas --with scipy python -c '...'` — MAD, trimmed SD, bootstrap CI for SD.

## Procedure

### Step 1 — Pick the right columns

For each candidate numeric column:
- Skip if it's an ID, code, or row index (monotonic increasing, all-unique integer). SD is meaningless.
- Skip if it's a boolean-coded 0/1 column unless the user asks (SD = sqrt(p(1-p)), rarely the useful summary).
- Include continuous measurements, counts, ratios, currency, scores.

Report any column you skipped and why.

### Step 2 — Compute the core statistics

For each column (and each group, if grouping):

| Statistic | What it tells you |
|---|---|
| `n` (non-null count) | Sample size the SD is based on. |
| `mean`, `median` | Centre. If they differ substantially, distribution is skewed. |
| `stddev_samp` | Standard deviation, n-1 denominator. **Default report value.** |
| `variance_samp` | Square of SD. Report if user explicitly wants it. |
| `min`, `max` | Range. Flag if max is >10× the 99th percentile — outlier pulling SD up. |
| `q25`, `q75`, `IQR` | Robust spread — compare to SD. |
| `mad` (median absolute deviation) | Robust SD analogue. MAD × 1.4826 ≈ SD if data is normal. |
| `cv` (coefficient of variation) = SD / mean | Dimensionless spread. Only meaningful when mean > 0 and the column has a natural zero (not temperatures-in-C). |

SQL sketch in DuckDB:

```sql
SELECT
  count(col)                                    AS n,
  avg(col)                                      AS mean,
  median(col)                                   AS median,
  stddev_samp(col)                              AS sd,
  quantile_cont(col, 0.25)                      AS q25,
  quantile_cont(col, 0.75)                      AS q75,
  quantile_cont(col, 0.75) - quantile_cont(col, 0.25) AS iqr,
  min(col)                                      AS min,
  max(col)                                      AS max
FROM '<file>';
```

### Step 3 — Diagnose whether SD is trustworthy

For each column, check:

- **n < 30**: SD estimate is unstable. Report with a warning and a bootstrap 95% CI if the user wants precision.
- **Skew > 1 or < -1**: the distribution is asymmetric; SD will overstate typical deviation. Prefer IQR or MAD.
- **Kurtosis > 5**: heavy tails; a few outliers dominate SD. Report trimmed SD (drop top/bottom 2.5%) alongside the raw SD.
- **SD ≈ 0 (relative to mean)**: column is near-constant. Check for an imputation spike or a pre-aggregated series.
- **|MAD × 1.4826 − SD| / SD > 0.3**: robust and classical spread disagree by >30%, meaning outliers are inflating SD. Flag.
- **mean ≈ 0 on CV request**: CV is undefined/unstable; don't report CV.

### Step 4 — Grouped / stratified (if grouping column provided)

Compute Step 2 statistics per group. Additionally surface:
- Groups with **substantially larger SD** than others (e.g. > 2× the median group SD). These may have different variance regimes or hidden subgroups.
- Groups with **n < 30** — SD not reliable.
- The ratio of between-group variance to within-group variance (a one-way ANOVA F would formalise this; for a quick read, compare `stddev(group_means)` to `mean(group_sds)`).

### Step 5 — Report

Write `outputs/standard-deviation/report.md`:

- **Summary table** of all reported columns with n, mean, median, SD, IQR, MAD, CV, and a `trust` flag (`ok` / `skewed` / `heavy-tailed` / `small-n` / `near-constant`).
- **Per-column notes** for anything flagged in Step 3.
- **Grouped table** (if applicable).
- **Interpretation hint** for each "ok" column: "≈68% of values fall within ±1 SD of the mean (assuming roughly normal)" — include this only for columns where the normality assumption is tenable.

Keep it concise. SD is a primitive; this skill should feed other skills, not produce a wall of prose.

## Notes

- Default to **sample** SD (n-1). Only use population SD when the user explicitly has the whole population (e.g. every country in a fixed list, not a sample).
- If the user asks for "standard deviation" of a categorical or string column, push back — they may want something else (frequency variance, entropy).
- Pairs with `multivariate-analysis` (which needs standardised inputs) and `forensic-sweep` (which uses SD patterns to detect tampering).
