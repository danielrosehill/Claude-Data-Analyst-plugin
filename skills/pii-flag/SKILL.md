---
name: pii-flag
description: Scan a dataset and flag columns or values that appear to contain personally identifiable information (PII). Use when the user wants a quick privacy audit of a CSV/Parquet/Excel file before sharing, publishing, or ingesting into another system.
---

# PII Flag

First-pass privacy scan of a dataset. Identifies columns likely to contain PII and samples matching rows.

## Inputs

- Path to a dataset file or folder.
- Optional: sensitivity level (`standard` default, `strict` also flags quasi-identifiers like ZIP, DOB, gender).

## Recommended CLI tooling

- `duckdb` — regex-based column scans at speed.
- `uv run --with presidio-analyzer python -c '...'` — Microsoft Presidio for ML-based entity detection when regex is insufficient.
- `ripgrep` (`rg`) — for ad-hoc text-file scans before structured analysis.

## Detection strategy

Run column-level checks in two passes:

### Pass 1: Name-based heuristics

Match column headers (case-insensitive) against PII vocabulary:
- **Direct**: `name`, `first_name`, `last_name`, `email`, `phone`, `mobile`, `address`, `street`, `ssn`, `nino`, `passport`, `national_id`, `credit_card`, `iban`, `account_number`, `dob`, `date_of_birth`
- **Quasi-identifiers** (strict mode): `zip`, `postcode`, `gender`, `ethnicity`, `age`

### Pass 2: Value-based regex on string columns

Sample up to 1000 rows per string column and test:
- Email: `[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}`
- Phone (loose): `\+?\d[\d\s\-()]{7,}\d`
- Credit card (Luhn-validated): 13–19 digits
- IPv4, IPv6
- Israeli ID (9 digits with checksum), US SSN (`\d{3}-\d{2}-\d{4}`)
- Free-text columns: run Presidio if available for PERSON, LOCATION, ORGANIZATION entities

## Output

Write `<dataset>-pii-report.md` containing:

| Column | Detection basis | Confidence | Sample matches (redacted) | Recommendation |

Confidence levels: **high** (regex + name match), **medium** (one of the two), **low** (value pattern only).

End the report with:
- Count of rows containing any PII
- Suggested remediation (drop column, hash, tokenize, pseudonymise, or redact)
- Callout if combining non-PII columns could re-identify individuals (k-anonymity concern)

**Never print raw PII values into the report** — always mask (e.g. `j***@example.com`, `***-**-1234`).
