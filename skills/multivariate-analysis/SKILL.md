---
name: multivariate-analysis
description: Test relationships among three or more variables simultaneously — partial correlations, controlled effects, multicollinearity, interaction terms, and dimensionality reduction. Use when a pairwise correlation sweep isn't enough and the user wants to know how variables behave together, which effects survive when others are held constant, and which clusters of variables move as one.
---

# Multivariate Analysis

Go beyond pairwise correlation. Tell the user which variables *actually* drive the target once the others are accounted for, which variables are redundant, and which combinations reveal structure that no single pair shows.

## When to use this vs. `correlation-analysis`

- `correlation-analysis` — "is A related to B?" Pairwise ranking.
- `multivariate-analysis` — "with A, B, C, D all in play, which matter, which are redundant, and are there interactions or latent factors?"

If the user says "correlation" but lists more than two variables of interest, or has a target they want to "explain", this is the right skill.

## Inputs

- Path to a dataset (CSV / Parquet / Excel / DuckDB table).
- Optional: target variable (the one to explain). If unset, do an unsupervised pass.
- Optional: candidate predictors. Default: all numeric columns + encoded low-cardinality categoricals.
- Optional: grouping / segment column (for stratified analysis).

## Recommended CLI tooling

- `duckdb` — correlation matrices, VIF precursors (regression residuals), standardisation.
- `uv run --with pandas --with scipy --with statsmodels --with scikit-learn python -c '...'` — partial correlation, VIF, PCA, factor analysis, OLS with interactions.

## Procedure

### Step 1 — Profile and prep

- Identify numeric columns. Encode low-cardinality categoricals (≤ ~20 levels) as one-hot or ordinal as appropriate.
- Report missingness per column. If >20% missing on any candidate, ask whether to drop the column, drop rows, or impute (median/mode or model-based).
- Standardise (z-score) numerics before anything scale-sensitive (PCA, regularised regression, distance-based methods).

### Step 2 — Full correlation matrix

Compute the full pairwise correlation matrix (Pearson by default; switch to Spearman if any variable is heavily skewed or ordinal). Surface:
- The strongest pairs (top 10 by |r|).
- Clusters of mutually correlated variables (simple hierarchical clustering on `1 - |r|`).

Save the matrix as `outputs/multivariate/corr_matrix.csv` and a heatmap if the user wants visuals.

### Step 3 — Partial correlations

For each pair of candidates (or each candidate vs. target), compute the **partial correlation** controlling for all other candidates. This is the key move: it tells you whether A↔B survives once C, D, E are held constant.

Report:
- Pairs whose correlation **strengthens** when controls are added (suppression).
- Pairs whose correlation **collapses** (confounded — the original pairwise r was riding on a third variable).
- Pairs that are **stable** (robust relationship).

### Step 4 — Multicollinearity check

Compute **Variance Inflation Factor (VIF)** for each predictor against the others.
- VIF < 5: fine.
- 5 ≤ VIF < 10: watch.
- VIF ≥ 10: redundant — flag for removal or combination.

Report the VIF table and name the minimal set of variables to drop to bring everything below 5.

### Step 5 — Targeted modelling (if target provided)

Fit three complementary models and compare:

1. **OLS with all candidates** — standardised coefficients, p-values, R². Shows which predictors matter when all compete.
2. **OLS with interaction terms** for the top 2–3 predictors (pairwise products). Shows whether effects depend on each other.
3. **Regularised regression** (Lasso / ElasticNet, 5-fold CV). Shows which predictors survive when the model is forced to be sparse.

Surface:
- Predictors significant in OLS but killed by Lasso → likely correlated with a stronger predictor.
- Significant interactions → note them; they'll be invisible to any purely pairwise analysis.
- Sign flips between simple correlation and OLS coefficient → **classic confounding**, highlight loudly.

### Step 6 — Dimensionality reduction (unsupervised)

Run PCA on the standardised numeric matrix. Report:
- How many components explain 80% / 95% of variance.
- The loadings for the first 3–5 components — which variables dominate each? Name the components if a theme is obvious ("size factor", "recency factor").
- Flag if a single component explains >60% of variance — the dataset has less independent information than the column count suggests.

Optionally: factor analysis if the user wants latent constructs rather than variance-maximising axes.

### Step 7 — Stratified pass (if grouping column provided)

Repeat Steps 2, 3, 5 within each group. Surface:
- Relationships that **reverse sign** across groups (Simpson's paradox candidate).
- Relationships that exist in one group and not another.

### Step 8 — Report

Write `outputs/multivariate/report.md` with:
- One-paragraph headline: which predictors actually drive the target, which are redundant, which interactions matter.
- The partial-correlation findings, the VIF list, the model comparison, the PCA summary.
- A "variables to drop" recommendation and a "variables to keep" recommendation with reasoning.
- Any Simpson's-paradox-style reversals found in Step 7.

## Notes

- Partial correlations and VIF both choke on perfect or near-perfect collinearity. If a computation fails, drop the worst offender (usually identifiable from the full correlation matrix) and retry.
- With n < 10 × number of predictors, model results are unstable — warn the user and prefer Lasso + cross-validation over plain OLS.
- Don't chase statistical significance alone; a 0.02 standardised coefficient with p = 0.001 in a 100k-row dataset is still practically negligible.
