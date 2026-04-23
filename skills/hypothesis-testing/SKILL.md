---
name: hypothesis-testing
description: Take a user-stated hypothesis and test it against the data, producing a report stating whether the data supports, refutes, or is inconclusive about the claim. Use when the user has a specific question or claim they want to interrogate against a dataset.
---

# Hypothesis Testing

Rigorous first-pass evaluation of a user hypothesis against a dataset.

## Inputs

- Path to a dataset file or folder.
- The **hypothesis**, stated in natural language (e.g. "customers in segment A spend more than those in segment B", "there's no difference in conversion by landing page").
- Optional: desired significance level α (default 0.05).

## Step 1 — Formalise the hypothesis

Before testing, restate the user's claim as:

- **H₀ (null)**: typically "no effect" / "no difference" / "no relationship".
- **H₁ (alternative)**: the user's claim, one- or two-sided as appropriate.
- **Variables involved**: identify which columns map to the hypothesis, with their dtypes.
- **Test type** selected based on the data shape (see decision table below).

Show this formalisation to the user and proceed unless they object.

## Step 2 — Select a test

| User hypothesis shape | Variables | Default test |
|---|---|---|
| Group A mean ≠ Group B mean | 1 numeric + 1 binary categorical | Welch's t-test (unequal var) or Mann-Whitney if non-normal |
| Difference across 3+ groups | 1 numeric + 1 categorical | One-way ANOVA or Kruskal-Wallis |
| Association between two categoricals | 2 categoricals | Chi-square (or Fisher's exact if small cells) |
| Correlation ≠ 0 | 2 numerics | Pearson (linear) or Spearman (monotonic) |
| Proportion differs from value | 1 binary | One-sample proportion z-test |
| Paired before/after | 1 numeric, paired | Paired t-test or Wilcoxon signed-rank |
| Time trend | 1 numeric + time | Mann-Kendall or regression slope test |

Check assumptions (normality via Shapiro-Wilk for small n or visual for large n; equal variance via Levene's) and fall back to the non-parametric counterpart when they fail.

## Step 3 — Execute

Recommended: `uv run --with pandas --with scipy --with statsmodels python -c '...'`.

Report:
- Sample size(s)
- Effect size (Cohen's d, r, Cramér's V, odds ratio — whichever fits)
- Test statistic and p-value
- 95% confidence interval for the effect

## Step 4 — Verdict

Classify as one of:

- **Supports** — p < α **and** effect size is non-trivial **and** sample size is adequate.
- **Refutes** — p < α but in the opposite direction, or a tight CI around zero effect.
- **Inconclusive** — p ≥ α, or underpowered (effect plausible but sample too small), or assumptions violated beyond repair.

Do **not** equate "p > 0.05" with "no effect" — distinguish "evidence of no effect" (tight CI around zero) from "no evidence of effect" (wide CI).

## Output

Write `<dataset>-hypothesis-<slug>.md`:

1. The hypothesis as stated by the user, verbatim.
2. Formalised H₀ / H₁.
3. Test chosen and why (including assumption checks).
4. Results table.
5. **Verdict** in bold: supports / refutes / inconclusive, with a plain-English explanation.
6. Caveats (confounders, selection effects, multiple-comparisons risk if this is one of many tests).
