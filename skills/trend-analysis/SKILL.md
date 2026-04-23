---
name: trend-analysis
description: Identify and report the major trends a dataset depicts — directional changes over time, growth rates, seasonal patterns, segment shifts, and emerging categories. Use when the user wants the headline "what is this data saying" narrative rather than a specific test.
---

# Trend Analysis

Identify the major trends in a dataset and summarise them in a narrative report.

## Inputs

- Path to a dataset file or folder.
- Optional: time column name (auto-detected if a single date/datetime column exists).
- Optional: metric(s) of interest — numeric columns to focus trend analysis on. Default: all numerics.
- Optional: segment column — to produce per-segment trend breakdowns.

## Recommended CLI tooling

- `duckdb` — windowed SQL aggregations (`time_bucket`, moving averages, YoY).
- `uv run --with pandas --with statsmodels python -c '...'` — STL decomposition, Mann-Kendall trend test, seasonal detection.
- `mlr` (miller) — quick pivots and tallies on CSV without loading pandas.

## Procedure

### Step 1 — Temporal trends (if time column exists)

For each metric:
- **Overall direction**: fit linear regression on the metric vs. time. Report slope, R², and sign (up/down/flat). Supplement with Mann-Kendall test for monotonic trend (robust to outliers and non-linearity).
- **Growth rate**: first-to-last period % change, and CAGR if span > 1 year.
- **Level shifts / change points**: detect structural breaks (PELT or simple rolling mean comparison).
- **Seasonality**: if ≥2 seasonal cycles present, run STL decomposition and report dominant period (weekly, monthly, yearly).
- **Recency**: compare last period vs. trailing average — is the trend accelerating, decelerating, or reversing?

### Step 2 — Compositional trends (categorical)

For each categorical column:
- Which categories are **growing** in share over time?
- Which are **shrinking**?
- Any **new entrants** appearing only in recent periods?
- Any categories that **disappeared**?

### Step 3 — Segment trends

If a segment column is provided, repeat Step 1 within each segment and surface:
- Segments with the strongest growth / decline
- Segments diverging from the overall trend (Simpson's-paradox watchlist)

### Step 4 — Non-temporal trends

If no time column exists, "trend" becomes distributional:
- Skew, concentration (top-N share, Gini), long-tail structure
- Dominant vs. rare categories
- Relationships between paired columns (lean on `correlation-analysis` for the heavy lifting; summarise headlines only)

## Output

Write `<dataset>-trends.md` structured as a **narrative**, not a data dump:

1. **Headline** — one sentence capturing the single most important trend.
2. **Key trends** — 3–7 bullets, each with: what's happening, direction, magnitude, and supporting number.
3. **Segment highlights** (if applicable) — who's driving and who's lagging.
4. **Anomalies in the trend** — single-period spikes, breaks, reversals worth investigating.
5. **Caveats** — short series, data gaps, segment-size issues that limit confidence.
6. **Chart recommendations** — name 2–4 specific charts that would best communicate these trends (don't generate images; name them).

The report should read like a briefing a busy stakeholder would understand in 60 seconds.
