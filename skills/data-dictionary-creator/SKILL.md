---
name: data-dictionary-creator
description: Generate a data dictionary for a dataset, combining automatic profiling with the user's description of what the data represents. Use when the user wants documentation of columns — names, types, semantic meaning, units, allowed values, and nullability — for a CSV/Parquet/Excel file.
---

# Data Dictionary Creator

Produce a data dictionary by merging schema inspection with the user's semantic description of the dataset.

## Inputs

- Path to a dataset file or folder.
- The user's description of the dataset: what it represents, how it was collected, what each column means (can be partial — infer the rest).
- Optional: output format (`markdown` default, `csv`, or `json`).

## Recommended CLI tooling

- `duckdb -c "DESCRIBE SELECT * FROM '<file>'"` — fast schema + inferred types.
- `csvstat` — null counts, uniqueness, min/max per column.
- `uv run --with pandas python -c '...'` — for dtype coercion and sampling.

## Procedure

### Step 1 — Auto-profile every column

For each column, collect:
- Inferred data type (int, float, string, date, boolean, category)
- Null count and percentage
- Unique count (and full value list if <20 distinct)
- Min / max / mean (numeric) or top-5 modes (categorical)
- Sample values (3 random non-null)

### Step 2 — Merge with the user's description

Parse the user's description and map sentences to columns. For each column, fill:

- **Name** (as in the file)
- **Display name** / human-readable label
- **Description** — one-sentence semantic meaning
- **Type**
- **Unit** (currency code, SI unit, %, count, etc.)
- **Allowed values** — enumerated list if categorical with small cardinality
- **Nullable** — yes/no and what a null means (missing vs. not-applicable)
- **Source** — where this field originated if the user mentioned it
- **PII** — none / direct / quasi-identifier (cross-check with `pii-flag` heuristics)
- **Notes** — caveats, known issues, derivation formulas

If a column isn't covered by the user's description, mark the Description field as `[NEEDS REVIEW]` rather than guessing, and list these at the end for user confirmation.

### Step 3 — Add dataset-level metadata

At the top of the dictionary:
- Dataset name and path
- Purpose (from user description)
- Row count, column count
- Primary key(s) — infer from uniqueness; ask if ambiguous
- Collection period if derivable from timestamp columns
- Last modified timestamp

## Output

Default — write `<dataset>-dictionary.md`:

```markdown
# Data Dictionary — <dataset name>

## Overview
...

## Columns

### `column_name`
- **Type**: ...
- **Description**: ...
- **Unit**: ...
- **Nullable**: ...
- **Allowed values**: ...
- **Sample**: ...
- **Notes**: ...
```

For `csv` output, flatten to one row per column with standard dictionary columns. For `json`, emit a structured schema object compatible with JSON Schema / Frictionless Data.

End with a `[NEEDS REVIEW]` section listing columns the user should clarify.
