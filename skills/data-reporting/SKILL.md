---
name: data-reporting
description: Produce a parametric PDF report describing a dataset — size, schema, distributions, key statistics, and findings from other skills — compiled via Typst. Use when the user wants a shareable, print-ready document about their data, not a one-off markdown summary.
---

# Data Reporting

Generate a parametric dataset report as a Typst-compiled PDF. "Parametric" here means the report structure is fixed and reusable — you fill in the parameters (dataset path, title, sections to include, cut-offs) and the document regenerates from scratch.

Use this skill when the user wants:
- A stakeholder-ready PDF describing a dataset.
- A reproducible document that can be regenerated on data refresh.
- A consolidation of outputs from other skills (`data-dictionary-creator`, `trend-analysis`, `correlation-analysis`, `standard-deviation`, `sample-size`, `forensic-sweep`, etc.) into one artefact.

For quick markdown summaries, use the reporting step inside the relevant analysis skill instead — this skill is for when PDF is the deliverable.

## Inputs

- Path to a dataset (CSV / Parquet / Excel / DuckDB table).
- Report parameters (all optional; sensible defaults applied):
  - `title` — report title. Default: dataset filename.
  - `subtitle` — e.g. snapshot date.
  - `author` — defaults to system user.
  - `sections` — which sections to include. Default: `overview`, `schema`, `size`, `distributions`, `quality`, `findings`.
  - `max_columns_profiled` — cap detailed profiling for very wide tables. Default: 50.
  - `include_charts` — boolean. Default: true.
  - `theme` — `plain` / `dsr-business` / `personal`. Default: `plain`.

## Recommended tooling

- `duckdb` — all stat computation.
- `uv run --with pandas --with matplotlib python -c '...'` — distribution plots rendered to PNG for Typst to embed. Matplotlib only (no seaborn) to keep dependencies minimal.
- `typst` — compile the document.
- The `typst-document-generator` skills (`public-doc`, `personal-doc`, `dsr-business-doc`) — for themed output. When `theme != plain`, delegate final compilation to the matching skill.

## Output layout

Create an output folder: `outputs/data-reporting/<dataset-stem>-<YYYYMMDD>/`:

```
outputs/data-reporting/sales-2026-04-23/
  report.typ          -- the Typst source
  report.pdf          -- compiled output
  assets/
    hist_price.png
    hist_quantity.png
    bar_category.png
    missingness.png
  data/
    summary_stats.csv
    schema.csv
  params.json         -- exact parameters used, for reproducibility
```

## Procedure

### Step 1 — Parse parameters

Resolve defaults. Validate that the dataset exists and is readable. Read row count and column count to calibrate the rest.

### Step 2 — Compute sections (as data)

For each requested section, produce the underlying tables / values. Don't render yet — keep everything as CSV / JSON under `data/` so the final compile is pure templating.

**overview** — dataset name, source path, row count, column count, file size, last modified, snapshot timestamp.

**schema** — per-column: name, inferred type, non-null count, null %, unique count, sample values. Cap to `max_columns_profiled`; note the cap in the report if hit.

**size** — delegate to `sample-size` skill (or inline the same computations): total n, unit of analysis (prompt the user if ambiguous), group cell sizes for any low-cardinality categorical.

**distributions** — for each numeric column: mean, median, SD, min, max, q25, q75, skew, kurtosis. For each categorical with ≤ 30 levels: top values with counts and %. Pull SD values from `standard-deviation` if already computed.

**quality** — null rates, duplicate row count, constant columns, suspected sentinel values. If `forensic-sweep` has been run, embed its headline verdict.

**findings** — include outputs from any prior skills the user has run (`trend-analysis`, `correlation-analysis`, `multivariate-analysis`, `hypothesis-testing`, `data-enrichment`). The user passes these in explicitly; don't rerun analyses.

### Step 3 — Render charts (if `include_charts`)

For each numeric column (up to `max_columns_profiled`): histogram PNG at `assets/hist_<col>.png`. 600×400 px, dpi=120, no title (Typst adds the caption).

For each included categorical: horizontal bar chart of top 15 levels at `assets/bar_<col>.png`.

One overall missingness chart: bar chart of null % per column at `assets/missingness.png`.

Keep the matplotlib script minimal — no styling flourishes. Typst handles presentation.

### Step 4 — Write `report.typ`

Emit a single Typst file. Structure:

```typst
#set document(title: "<title>", author: "<author>")
#set page(paper: "a4", margin: 2cm, numbering: "1 / 1")
#set text(font: "New Computer Modern", size: 10pt)
#set heading(numbering: "1.1")

#align(center)[
  #text(size: 20pt, weight: "bold")[<title>]
  #v(0.3em)
  #text(size: 12pt)[<subtitle>]
  #v(0.3em)
  #text(size: 10pt)[<author> · <snapshot-date>]
]

#outline()

= Overview
<overview prose + small table>

= Schema
#table(
  columns: (auto, auto, auto, auto, auto),
  [*Column*], [*Type*], [*Non-null*], [*Unique*], [*Sample*],
  ...rows...
)

= Sample Size
<size section>

= Distributions
== Numeric columns
<for each: summary row + histogram figure>

== Categorical columns
<for each: top-values table + bar figure>

= Data Quality
<null rates table, duplicate count, sentinel flags, forensic-sweep verdict if present>

= Findings
<embedded prior-skill outputs, one subsection each>

= Reproducibility
Generated <date> from `<dataset-path>`.
Parameters: see `params.json`.
```

Use `#figure(image("assets/hist_price.png"), caption: [Distribution of price])` for charts. Use `#table(...)` for tabular data. Keep column widths readable — no tables wider than the page.

### Step 5 — Compile

If `theme == plain`:
```bash
typst compile report.typ report.pdf
```

If `theme` is `dsr-business`, `personal`, or `public`: delegate to the matching `typst-document-generator` skill, passing the generated body. Those skills have the themed templates already wired; don't reinvent styling.

Verify the PDF exists and is non-trivially sized (> 10 KB).

### Step 6 — Save parameters

Write `params.json` with the exact inputs (dataset path, absolute; all parameters resolved; skill version). This is what makes the report reproducible.

### Step 7 — Report back

Tell the user:
- Path to the PDF.
- Path to the Typst source (for edits).
- How to regenerate (`typst compile outputs/.../report.typ`).
- Any sections that were skipped (e.g. "distributions skipped for 12 columns over the cap — raise `max_columns_profiled` to include them").

## Notes

- **Typst must be on PATH.** If it isn't, fall back to markdown output and tell the user how to install (`cargo install typst-cli` or the release binary from typst.app).
- Wide tables: if a table would overflow the page width, split it by column groups or use landscape for that section only (`#page(flipped: true)[ ... ]`).
- Don't put raw data rows in the PDF — aggregates, samples, and schema only. Full data belongs in the source files.
- Pair with `data-dictionary-creator` for the schema section (use its output as-is) and `sample-size` / `standard-deviation` for the stat blocks — avoid re-implementing those computations here.
- If the user wants the report hosted or shared, don't auto-upload. Hand them the PDF path and let them move it.
