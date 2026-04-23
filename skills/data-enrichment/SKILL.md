---
name: data-enrichment
description: Identify what the user is trying to analyse, diagnose gaps in the current dataset, propose external data sources that could fill them, then plan and implement the enrichment. Use when the dataset alone can't answer the user's question and extra context (reference data, lookups, joinable public datasets) is needed.
---

# Data Enrichment

Turn an under-powered dataset into one that can actually answer the user's question, by identifying gaps and fusing in external data.

## Inputs

- Path to the primary dataset (CSV / Parquet / Excel / DuckDB table).
- The user's analytical goal — what question are they trying to answer? If not stated, ask.
- Optional: constraints (offline only, no paid APIs, must be open-data-licensed, etc.).

## Procedure

### Step 1 — Understand the goal

Restate the user's question in one sentence. Identify the **analytical unit** (row = customer? transaction? country-year?) and the **target** (what are we trying to explain, predict, compare, or rank?).

If the goal is vague ("analyse this data"), push back: ask what decision or insight they want. Enrichment without a target is busywork.

### Step 2 — Profile what's already there

Run a quick schema + sample on the dataset:

```bash
duckdb -c "DESCRIBE SELECT * FROM '<file>'"
duckdb -c "SELECT * FROM '<file>' LIMIT 5"
```

Note the columns grouped by role:
- **Identifiers / join keys** — IDs, codes, names, dates, locations (these are the hooks for enrichment).
- **Dimensions** — categories, segments.
- **Measures** — the numeric columns the user will want to explain.
- **Time** — any date/datetime columns.

### Step 3 — Diagnose gaps

Compare the data to the goal and list concrete gaps. Each gap should name a **missing variable** or **missing context**, not just "more data". Examples:

- Goal: "Why did Q3 sales drop?" — dataset has sales but no **marketing spend**, **weather**, **competitor pricing**, or **macro indicators** for that period.
- Goal: "Which customers are highest value?" — dataset has transactions but no **customer demographics** or **acquisition channel**.
- Goal: "Compare our countries' performance fairly" — raw numbers exist but no **population**, **GDP**, or **currency conversion** to normalise by.

For each gap, note: what variable is missing, why it matters for the goal, and what join key would connect it (country code, date, customer ID, postcode, ...).

### Step 4 — Propose sources

For each gap, propose 1–3 candidate external sources. Evaluate each on:

| Criterion | What to check |
|---|---|
| **Joinability** | Does it share a key with the primary dataset? (ISO codes, dates, lat/lon, ids) |
| **Coverage** | Does it span the time range / geography / population of the primary data? |
| **Freshness** | How current is it? |
| **Licence** | Is it redistributable? (ODbL, CC-BY, public domain, commercial...) |
| **Access** | Bulk download, API, scrape? Cost? Rate limits? |
| **Granularity** | Does the resolution match? (country-year vs. city-month vs. postcode-day) |

Common reliable sources (use WebSearch / WebFetch to verify current endpoints before implementing):
- **Geo / admin**: Natural Earth, GeoNames, OpenStreetMap Nominatim, country-code lookup tables.
- **Economic / demographic**: World Bank Open Data, IMF, OECD, Eurostat, national statistics offices.
- **Weather**: Open-Meteo (free, no key), NOAA, ECMWF.
- **FX / commodities**: ECB reference rates, Frankfurter API.
- **Company / business**: OpenCorporates, Companies House (UK), SEC EDGAR.
- **General reference**: Wikidata SPARQL, DBpedia.

Present the source shortlist to the user. Let them pick — don't silently fan out to 5 APIs.

### Step 5 — Plan the enrichment

Before writing code, write the plan as a short table:

| Target column(s) | Source | Join key | Method |
|---|---|---|---|
| `population`, `gdp_usd` | World Bank API | `iso3` + `year` | HTTP fetch + join |
| `avg_temp_c` | Open-Meteo archive API | `lat`,`lon` + `date` | HTTP fetch per location + join |
| `currency_to_usd` | Frankfurter API | `currency` + `date` | HTTP fetch + join |

Flag: cardinality of API calls (one per row? one per unique key?), caching strategy, expected runtime.

Get user sign-off on the plan before implementing — especially if it involves paid APIs or thousands of requests.

### Step 6 — Implement

Write the enrichment as a reproducible script under `scripts/enrich_<topic>.py` (or `.sql` if it's pure DuckDB + HTTP extension). Conventions:

- **Cache raw API responses** to `data/enrichment/cache/` keyed by query params. Re-runs should be free.
- **Stage enriched outputs** to `data/enrichment/<source>.parquet` — don't mutate the primary dataset in place.
- **Join at query time** via a DuckDB view, or materialise a `*_enriched` table if joins are expensive.
- **Log provenance**: add a row to `data/enrichment/PROVENANCE.md` with source URL, fetch date, licence, and a SHA256 of the cached payload.

Example structure:

```
data/
  raw/
    sales.csv
  enrichment/
    cache/
      worldbank_population_IND_2015-2024.json
      openmeteo_40.71_-74.01_2024.json
    worldbank_indicators.parquet
    weather_daily.parquet
    PROVENANCE.md
scripts/
  enrich_worldbank.py
  enrich_weather.py
```

### Step 7 — Verify

After enrichment:
- **Join coverage**: what % of primary rows matched? Report unmatched keys (common causes: country name vs. ISO code mismatch, date timezone drift, trailing whitespace in IDs).
- **Sanity checks**: do the enriched values look reasonable? (population not negative, temperatures in expected range, FX rates within ±20% of known values).
- **Re-run the original question** against the enriched dataset. Did the gap actually close? If not, diagnose why before adding more sources.

### Step 8 — Document

Update the repo's `CLAUDE.md` (or create a `data/enrichment/README.md`) with:
- What enrichments exist and what question they were added to answer.
- How to refresh them (`python scripts/enrich_weather.py`).
- Join keys and any gotchas (e.g. "Kosovo uses XKX not ISO, manually mapped").

## Notes

- **Don't enrich speculatively.** Every added column is maintenance. Only pull what the stated goal requires.
- **Respect rate limits.** Space API calls; use the cache aggressively.
- **Licence matters if the output leaves the machine.** Check before the user publishes a derived dataset.
- If a gap has no good public source, say so — "can't be closed with open data, would need a commercial provider or primary research" is a valid outcome.
- If the dataset is already loaded via `setup-data-workspace`, attach enrichments as new tables in the same `.duckdb` and update the Data section of CLAUDE.md.
