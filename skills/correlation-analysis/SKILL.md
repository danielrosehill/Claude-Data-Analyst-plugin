---
name: correlation-analysis
description: Detect and compute correlations between numeric variables in a dataset. Use when the user wants to see how variables in a CSV/Parquet/Excel file move together — Pearson, Spearman, or Kendall — with a short report flagging the strongest positive and negative pairs.
---

# Correlation Analysis

Produce a first-pass correlation report for a dataset in a folder.

## Inputs

- Path to a dataset file (CSV, TSV, Parquet, XLSX) or folder containing one.
- Optional: correlation method (`pearson` default, `spearman` for non-linear/ranked, `kendall` for small-n or many ties).
- Optional: target variable — if given, rank all other numeric columns by absolute correlation to it.

## Recommended CLI tooling

- `duckdb` — fastest way to load mixed formats and run `CORR(x, y)` in SQL.
- `uv run --with pandas --with scipy python -c '...'` — for Spearman/Kendall and heatmap export.
- `csvstat` (csvkit) — quick column types and null counts before correlating.

## Procedure

1. **Load and profile**: identify file format, row count, and column dtypes. Drop or flag non-numeric columns.
2. **Null/variance screen**: exclude columns with >50% nulls or zero variance — note them in the report.
3. **Compute correlation matrix** using the chosen method.
4. **Rank pairs** by absolute correlation. Report:
   - Top 5 positive (r > 0.3)
   - Top 5 negative (r < -0.3)
   - Any suspicious pairs (|r| > 0.95 — likely duplicates or derived columns)
5. **Caveats**: correlation is not causation; check for confounders; Pearson assumes linearity; small-n results are unreliable (flag if n < 30).

## Output

Write a markdown report next to the dataset (`<dataset>-correlations.md`) with:

- Method used and sample size
- Ranked pair table
- Flagged column exclusions
- One-paragraph plain-English summary of the strongest relationships

If a `--target` was given, lead with a ranked list of predictors of that target.
