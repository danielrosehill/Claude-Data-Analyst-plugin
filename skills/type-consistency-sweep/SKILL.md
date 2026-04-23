---
name: type-consistency-sweep
description: Scan one or more datasets for data-type inconsistencies that would block analysis or relational/graph database loading — mixed types within a column, the same logical field typed differently across files, string-encoded numbers/dates, inconsistent null sentinels. Report findings, and either delegate the fix to a Claude-Data-Wrangler skill or apply small wrangling in place.
---

# Type Consistency Sweep

Hybrid analysis + wrangling. Find the type inconsistencies that silently break joins, skew aggregations, and cause `COPY INTO` failures when the user tries to load the dataset into Postgres / DuckDB / BigQuery / a graph store. Then fix them — directly for trivial cases, or by handing off to the right specialist skill.

## Inputs

- Path to a dataset file, folder, or DuckDB database.
- Optional: intended destination (`postgres`, `duckdb`, `bigquery`, `neo4j`, `parquet`, `none`) — affects strictness.
- Optional: whether the user wants fixes applied, or just the report. Default: report first, then ask.

## Scope of inconsistencies to detect

### Within a single column

- **Mixed types**: column inferred as `VARCHAR` but >80% of values parse as numeric — a few stray strings are poisoning the type.
- **Stringified numbers**: `"1,234"`, `"$19.99"`, `"3.14 "` — numeric intent, string storage.
- **Stringified dates**: dates held as strings with inconsistent formats (`2024-01-15`, `15/01/2024`, `Jan 15 2024` all in one column).
- **Stringified booleans**: `"Y"`/`"N"`, `"true"`/`"false"`, `"1"`/`"0"`, `"yes"`/`"no"` — sometimes mixed.
- **Inconsistent null sentinels**: mix of actual NULL, empty string, `"NA"`, `"N/A"`, `"null"`, `"-"`, `"-999"`.
- **Integer stored as float** with all-zero fractional parts (`1.0`, `2.0`, ...) — will break joins to an integer key.
- **Unicode / encoding drift**: same logical value with different whitespace, case, or unicode form (NFC vs NFD).

### Across columns / files

- Same logical field (customer_id, country_code, date) **typed differently** in different files or tables — one file has it as `INTEGER`, another as `VARCHAR`, a third zero-padded.
- Join keys where one side is `"001"` and the other is `1`.
- Date/time columns with **inconsistent timezone handling** across files.
- Currency columns without a currency code, stored as plain numbers, and differing across files.

### Relative to destination

- Destination is **Postgres / BigQuery**: flag VARCHAR columns that should be `NUMERIC`/`DATE`/`BOOLEAN`; nested JSON in a column if destination is relational and not JSON-aware.
- Destination is **graph store (Neo4j, etc.)**: flag inconsistent node-id types across files that would prevent relationship creation.
- Destination is **Parquet / Arrow**: flag mixed-type columns (Arrow requires typed columns).

## Procedure

### Step 1 — Inventory

List every file/table in scope. For each column, capture:
- Inferred type (`duckdb -c "DESCRIBE SELECT * FROM '<file>'"`).
- Sample of non-null distinct values.
- Null / sentinel distribution.
- Whether >5% of values *fail* to parse under the inferred type.

### Step 2 — Detect within-column issues

For each column, run the checks above. For numeric/date candidates held as strings, try parsing (`TRY_CAST`, `strptime`) and record pass rate. Anything below ~95% clean parse is a finding.

### Step 3 — Detect cross-column / cross-file issues

Match columns by name (fuzzy: lowercase, strip separators) across files. For each match group, compare types and sample values. Flag mismatches.

### Step 4 — Classify severity

| Severity | Meaning |
|---|---|
| `blocker` | Will fail load into destination, or will silently corrupt joins/aggregations. |
| `warning` | Works, but introduces ambiguity or subtle bugs (e.g. string `"1.0"` comparisons). |
| `cosmetic` | Stylistic, not analytically harmful (trailing whitespace in a free-text column). |

### Step 5 — Decide the fix route

For each finding, pick one of:

**(a) Fix in place** — when the change is small, safe, reversible, and fully specified. Examples:
- Trimming whitespace in a single column.
- Casting a column of `"1.0"`, `"2.0"`, ... to `INTEGER`.
- Normalising a sentinel like `"-999"` to NULL.
- Unifying boolean encodings in one column.

Apply via DuckDB: write the result to a new file (`*_typed.parquet`) or a new table, never overwrite the source. Log the transformation.

**(b) Delegate to a `Claude-Data-Wrangler` skill** — when the fix matches an existing specialist skill, invoke it rather than re-implementing. Matching map:

| Finding | Delegate to |
|---|---|
| Text-formatted numbers across a file | `Claude-Data-Wrangler:text-to-numeric` |
| Mixed / inconsistent date formats | `Claude-Data-Wrangler:date-wrangling` |
| Country name vs ISO code mismatch | `Claude-Data-Wrangler:standardise-country-names` or `:add-iso3166` |
| Missing currency codes on monetary columns | `Claude-Data-Wrangler:enrich-with-currency` |
| Unicode / case / whitespace drift | `Claude-Data-Wrangler:unicode-consistency` |
| JSON needs flattening for relational load | `Claude-Data-Wrangler:json-restructure` |
| CSV ↔ JSON conversion for destination | `Claude-Data-Wrangler:csv-to-json` |
| Loading into a SQL / graph database | `Claude-Data-Wrangler:sql-load` or `:graph-database` |
| Cleanliness audit beyond types | `Claude-Data-Wrangler:data-cleanliness-scan` |

Delegation means invoking the matching skill via `Skill` with the relevant file path and options. Don't silently re-do the work.

**(c) Escalate to the user** — when the fix requires a judgement the agent shouldn't make alone. Examples:
- Deciding which of two conflicting type interpretations is canonical.
- Dropping rows vs. imputing vs. leaving NULL.
- Choosing a timezone for naive timestamps.

### Step 6 — Report

Write `outputs/type-consistency-sweep/report.md`:

- **Headline**: is the dataset load-ready for the stated destination? `yes` / `yes with warnings` / `no — blockers`.
- **Findings table**: file, column, issue, severity, recommended route (fix-in-place / delegate-to-<skill> / ask-user), status (`pending` / `fixed` / `delegated` / `skipped`).
- **Changes applied**: list of fix-in-place transformations with before/after samples.
- **Delegated tasks**: list of wrangler skills invoked and their outputs.
- **Open questions**: anything escalated to the user.

### Step 7 — Re-sweep (after fixes)

After any fixes or delegated wrangling, re-run Steps 1–3 on the output artifacts. Confirm blockers are resolved. Update the report with the post-fix state.

## Notes

- **Never overwrite source files.** Write typed outputs alongside originals (`*_typed.parquet`, `*_normalised.csv`) and make the new path visible in the report.
- For very large datasets, sample the first 100k rows for detection, but apply fixes to the full file.
- If the `Claude-Data-Wrangler` plugin is not installed, fall back to fix-in-place and note in the report that a wrangler skill would have been the better path.
- Pairs well with `forensic-sweep` (detects whether cleaning has happened) — type-consistency-sweep fixes what's broken, forensic-sweep flags what's been changed without permission.
