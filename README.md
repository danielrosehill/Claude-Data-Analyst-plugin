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
