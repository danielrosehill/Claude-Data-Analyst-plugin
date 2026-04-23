# Claude Data Analyst

First-pass data analysis toolkit for Claude Code. Point it at a CSV, Parquet, or Excel file and get an initial impression — correlations, PII audit, anomalies, hypothesis checks, a data dictionary, or a trend narrative.

## Skills

| Skill | What it does |
|---|---|
| `correlation-analysis` | Compute Pearson/Spearman/Kendall correlations and rank the strongest variable pairs. |
| `pii-flag` | Scan columns and values for likely PII; mask samples; recommend remediation. |
| `anomaly-analysis` | Three-layer anomaly sweep: value sanity, distribution outliers, multivariate/temporal. |
| `hypothesis-testing` | Formalise a user-stated hypothesis, pick the right test, and return supports/refutes/inconclusive. |
| `data-dictionary-creator` | Merge auto-profiled schema with the user's description into a full data dictionary. |
| `trend-analysis` | Identify and narrate the major trends — directional, seasonal, compositional, per-segment. |
| `setup-data-workspace` | Discover data files in the current repo, load them into a DuckDB database, and update CLAUDE.md with query instructions. |
| `data-enrichment` | Diagnose gaps between the user's analytical goal and the dataset, propose external sources, plan and implement enrichment. |
| `multivariate-analysis` | Partial correlations, VIF, regression with interactions, Lasso, and PCA to tell which variables actually drive the target and which are redundant. |
| `forensic-sweep` | Flag data that looks suspiciously clean, imputed, smoothed, or pre-normalised — so the user knows what was done upstream before they got it. |
| `type-consistency-sweep` | Detect within- and cross-file type inconsistencies that block analysis or DB loading; fix trivial cases or delegate to a `Claude-Data-Wrangler` skill. |
| `standard-deviation` | Compute SD (plus variance, IQR, MAD, CV) for numeric columns with trustworthiness flags for skew, heavy tails, and small n. |
| `sample-size` | Characterise the *effective* sample size per analytical question, flag underpowered segments, and give a go/no-go verdict. |
| `data-reporting` | Generate a parametric PDF report (Typst) describing the dataset — schema, distributions, quality, findings from prior skills. |

## Recommended CLI tooling

The skills assume (and will suggest) these are available on `PATH`:

- [`duckdb`](https://duckdb.org/) — SQL over CSV/Parquet/Excel at speed.
- [`csvkit`](https://csvkit.readthedocs.io/) — `csvstat`, `csvcut`, `csvlook`.
- [`miller`](https://miller.readthedocs.io/) (`mlr`) — pivots and tallies on CSV.
- [`uv`](https://docs.astral.sh/uv/) — run pandas/scipy/statsmodels/scikit-learn one-liners without a persistent venv.

Optional:

- [`presidio-analyzer`](https://microsoft.github.io/presidio/) — ML-backed PII entity detection (via `uv run --with presidio-analyzer`).

## Installation

```bash
claude plugins install claude-data-analyst@danielrosehill
```

## License

MIT.
