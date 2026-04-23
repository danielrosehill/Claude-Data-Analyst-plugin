---
name: anomaly-analysis
description: Scan a dataset for significant anomalies — outliers, distribution shifts, impossible values, and unusual groupings. Use when the user wants a first-pass integrity and anomaly sweep of a CSV/Parquet/Excel file before deeper analysis.
---

# Anomaly Analysis

Identify significant anomalies in a dataset across three layers: value-level, distribution-level, and relational.

## Inputs

- Path to a dataset file or folder.
- Optional: timestamp column name (enables temporal anomaly checks).
- Optional: group-by column (for per-segment anomaly detection).

## Recommended CLI tooling

- `duckdb` — percentile, z-score, and windowed queries.
- `uv run --with pandas --with scikit-learn python -c '...'` — IsolationForest and LOF for multivariate anomalies.
- `csvstat` (csvkit) — quick min/max/null sanity check.

## Procedure

### Layer 1 — Value-level sanity

For each column:
- Nulls: count and percentage; flag columns >20% null.
- Duplicates: flag rows duplicated on a natural key, or full-row duplicates.
- Impossible values: negative ages, dates in the future, percentages >100, etc. Use column name hints.
- Type coherence: mixed types in one column (e.g. numbers stored as strings with stray text).

### Layer 2 — Distribution-level outliers

For each numeric column:
- **IQR method**: flag values below Q1 − 1.5·IQR or above Q3 + 1.5·IQR.
- **Z-score**: flag |z| > 3.
- **Top/bottom 5**: list the extreme values explicitly so the user can eyeball them.

For categorical columns:
- Rare categories (<1% frequency) — possible typos or data-entry errors.
- Unexpected values outside a known vocabulary (if provided).

### Layer 3 — Multivariate and temporal

- **IsolationForest** on numeric columns → flag rows in the top 1% anomaly score.
- If a timestamp column exists: detect gaps, spikes, and level shifts in row volume over time.
- If a group-by column is provided: re-run Layer 2 within each group — an anomaly in-group may not be one globally.

## Output

Write `<dataset>-anomalies.md`:

1. **Summary**: one-line severity (low / medium / high) and headline anomaly count.
2. **By layer**: sections for value-level, distribution, and multivariate.
3. **Evidence**: concrete example rows (with row numbers / keys) for each flagged anomaly.
4. **Recommendations**: which anomalies warrant investigation vs. are likely expected tail behaviour.

Be specific — "17 rows have negative `order_total`" is useful; "there are some outliers" is not.
