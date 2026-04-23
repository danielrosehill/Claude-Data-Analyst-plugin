---
name: forensic-sweep
description: Scan a dataset for signs that it has been pre-cleaned, normalised, imputed, smoothed, deduplicated, or otherwise processed before the user received it — data that is "suspiciously clean". Flag findings so the user knows whether they're analysing raw reality or someone else's editorial choices.
---

# Forensic Sweep

Real-world data is messy. Clean data is often a clue that something upstream made it that way — and that "something" shapes any conclusion downstream. This skill sweeps for those tells and reports them so the user can decide whether the cleaning is legitimate (a curated public dataset) or misleading (aggregation that hides the interesting variance).

## Inputs

- Path to a dataset (CSV / Parquet / Excel / DuckDB table).
- Optional: known provenance (where the data came from, any processing steps the user is aware of). Helps calibrate suspicion.

## Recommended CLI tooling

- `duckdb` — distinct counts, null counts, distribution quantiles, digit-frequency tests.
- `uv run --with pandas --with scipy python -c '...'` — Benford's law, Shapiro/KS normality, duplicate-run detection.

## What to look for

### 1. Impossible tidiness

- **Zero nulls everywhere** in a dataset that represents a real-world process that normally has some missingness (surveys, logs, transactions). Real data almost always has *some* nulls. Total absence suggests imputation or a dropna upstream.
- **No duplicates at all** in transactional or event data where repeats are natural.
- **All strings trimmed, lowercased, single-spaced** uniformly — evidence of a normalisation pipeline.
- **Dates all in the same exact format, no parsing errors** — a cleaner has been through.

### 2. Imputation fingerprints

- A **spike** at the median, mean, or mode of a column (frequency of one value disproportionate to the rest of the distribution).
- A **spike at zero** in a column where zero is a plausible imputation placeholder but not a natural value.
- Values like `-999`, `-1`, `9999`, `N/A`, `UNKNOWN`, `MISSING` appearing frequently — sentinel values for missingness.
- Categorical columns with an "Other" / "Unspecified" bucket at suspiciously high share (>15%).

### 3. Smoothing / aggregation

- Numeric distributions that are **too normal** — run Shapiro-Wilk or Kolmogorov-Smirnov vs. the fitted normal. Real-world measurements are rarely textbook-normal.
- **Low kurtosis** and **no outliers at all** (e.g. all values within 2σ of the mean). Real data has tails.
- Time series with **implausibly smooth trajectories** — check first differences; if they're tightly distributed with no spikes, suspect a moving average or spline fit.
- Rounded numbers in clusters (e.g. everything to the nearest 5 or 10) — rounding or binning upstream.

### 4. Benford's-law violation (for naturally-occurring numeric data)

- For columns representing naturally-occurring magnitudes (prices, populations, revenues, counts), compare the distribution of leading digits to Benford's expected frequencies.
- Strong deviation (χ² p < 0.01) is a flag — could be fabrication, could be rounded/capped data, could be legitimately bounded (uniform IDs, assigned ranges). Interpret with context.

### 5. Deduplication / sampling traces

- Row counts that are **suspiciously round** (exactly 10,000; exactly 1,000,000). Real extracts rarely land on round numbers.
- IDs that form a **continuous sequence with no gaps** in a domain where gaps are expected (deletions, failed transactions).
- Stratified-sample fingerprints: every category has **exactly** the same row count.

### 6. Normalisation / scaling

- Numeric columns where **min ≈ 0 and max ≈ 1** → min-max scaled.
- Numeric columns where **mean ≈ 0 and std ≈ 1** → z-score standardised.
- Numeric columns with integer codes 0/1/2/... in a column that should be continuous → binned or quantised.

### 7. Typographic uniformity

- Free-text columns where every value matches one of a small closed set — the text has been mapped to a controlled vocabulary.
- No typos, no case variation, no whitespace variation in fields that in raw form always have them (city names, company names, email-entered fields).

### 8. Temporal tells

- A dataset claiming to cover "2020–2024" where all records are timestamped at midnight, or all on the first of the month → timestamps have been truncated/aggregated.
- No records at all from expected quiet periods (e.g. no weekend transactions in a retail dataset) — either real, or filtered.

## Procedure

### Step 1 — Profile

Gather basic stats per column: null %, unique count, min, max, mean, std, top-5 values with frequencies, skew, kurtosis. This is the substrate for everything else.

### Step 2 — Run each check

For each category above, run the relevant test. Record each finding with:
- **Column(s) affected.**
- **Severity**: `low` (plausible, worth noting), `medium` (unusual, ask the user), `high` (strong evidence of processing).
- **Evidence**: the specific statistic or pattern (e.g. "37% of values equal 0, next most common is 2.4%").
- **Most likely cause(s)**: what upstream process would produce this signature.
- **Implication**: what this means for downstream analysis (e.g. "regression coefficients on this column will be biased toward zero because the imputed mean dilutes variance").

### Step 3 — Cross-column check

- If *every* numeric column has mean ≈ 0 and std ≈ 1, it's not one column being z-scored — the whole table was.
- If multiple columns share a null pattern at the exact same rows, they were dropped/kept together — the cleaner made row-level decisions, not column-level.

### Step 4 — Report

Write `outputs/forensic-sweep/report.md`:

- **Headline verdict**: `likely raw` / `lightly cleaned` / `heavily processed` / `cannot tell`.
- **Findings table**, sorted by severity.
- **Recommended next steps** for the user — which findings warrant going back to the source, which can be safely documented and ignored, which should change how they model the data.

If severity is `high` on anything, say so at the top — don't bury it.

## Notes

- Curated public datasets (World Bank, Eurostat, ONS) are expected to be clean — "suspiciously clean" here means "clean in ways the user didn't expect".
- Always ask about provenance before declaring fraud or fabrication. Most "suspicious" signals have innocent explanations (ETL pipelines, privacy redaction, vendor pre-processing).
- Pair with `data-dictionary-creator` if the user wants to document each column's processing state.
- This skill **reports**, it does not **undo** processing. For fixing type inconsistencies, see `type-consistency-sweep`.
