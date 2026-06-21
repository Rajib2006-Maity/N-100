# N-100
SPRINT 1 — DATA FOUNDATION
# nifty100-etl — Sprint 1: Data Foundation

A SQLite-backed ETL pipeline that loads 12 source Excel files (7 core +
5 supplementary) covering 92 NIFTY100-style companies into a normalised
11-table database (`nifty100.db`), validated by 16 automated data
quality (DQ) rules and backed by a 100+ test pytest suite.

> **Synthetic data notice:** no real NIFTY100 source files were
> supplied for this build. `src/etl/generate_synthetic_data.py`
> generates a realistic 92-company demo dataset (12 Excel files,
> matching column structure) so the entire pipeline — loader,
> normaliser, validator, ratios calculator, tests — runs end-to-end
> and is fully demonstrable. **To use real data**, drop the 12 real
> NIFTY100 export files into `data/raw/` with the same filenames and
> columns (see `src/etl/loader.py` for the exact expected schema per
> file) and re-run `make load`; the rest of the pipeline is unchanged.

## Quick start

```bash
make setup      # create .venv and install dependencies
make load       # generate synthetic data (if data/raw/ is empty) + full load
make ratios     # compute financial_ratios from loaded P&L/BS/CF data
make validate   # run all 16 DQ rules, write output/validation_failures.csv
make test       # run the full pytest suite
make report     # one-shot: load + ratios + validate, printed summary
make clean      # remove db, generated data, and output artifacts
```

Or run the full pipeline from a clean checkout in one shot: `make all`.

## Project structure

```
nifty100-etl/
├── db/schema.sql                  11-table schema, PK/FK constraints, indexes
├── data/raw/                      12 source Excel files (synthetic demo data)
├── data/processed/                 reserved for any intermediate exports
├── output/                          load_audit.csv, load_audit_summary.txt,
│                                      validation_failures.csv
├── notebooks/exploratory_queries.sql  10 example analytical queries
├── docs/day06_manual_review.md         manual review write-up (5 flagged companies)
├── src/etl/
│   ├── normaliser.py               year/ticker/numeric/URL normalisation helpers
│   ├── loader.py                    reads 12 xlsx files, loads into nifty100.db
│   ├── validator.py                  16 DQ rules (DQ-01 .. DQ-16)
│   ├── compute_ratios.py              derives financial_ratios from loaded data
│   └── generate_synthetic_data.py      builds the 92-company demo dataset
├── tests/etl/
│   ├── test_normaliser.py           35 tests
│   ├── test_loader.py                 21 tests
│   └── test_validator.py               46 tests
├── Makefile
├── requirements.txt
└── .env.example                     configurable paths + DQ tolerances
```

## Database schema (11 tables)

`sectors`, `companies`, `profitandloss`, `balancesheet`, `cashflow`,
`stock_prices`, `financial_ratios`, `analysis`, `documents`,
`prosandcons`, `peer_groups` — all child tables reference `companies`
(and `companies` references `sectors`) with `PRAGMA foreign_keys = ON`
enforced by the loader. `financial_ratios` is the one table *not*
loaded from a raw file: it's computed by `make ratios` from already-
loaded P&L/BS/CF data (ROCE, ROE, debt-to-equity, current ratio).

## Source file → table mapping

| Source file | Target table |
|---|---|
| sectors.xlsx | sectors |
| companies_master.xlsx | companies |
| profit_and_loss.xlsx | profitandloss |
| balance_sheet.xlsx | balancesheet |
| cash_flow.xlsx | cashflow |
| stock_prices.xlsx | stock_prices |
| analysis_notes.xlsx | analysis |
| shareholding_pattern.xlsx | analysis |
| corporate_actions.xlsx | analysis |
| company_documents.xlsx | documents |
| pros_and_cons.xlsx | prosandcons |
| peer_groups.xlsx | peer_groups |

## The 16 data quality rules

| Rule | Severity | Checks |
|---|---|---|
| DQ-01 | CRITICAL | Primary key uniqueness on every table |
| DQ-02 | CRITICAL | Composite (company_id, year) uniqueness on yearly tables |
| DQ-03 | CRITICAL | Foreign key integrity (`PRAGMA foreign_key_check`) |
| DQ-04 | WARNING | Balance sheet total assets ≈ total liabilities (1% tolerance) |
| DQ-05 | WARNING | Stored OPM% vs. computed OPM% cross-check |
| DQ-06 | WARNING | Sales must be positive |
| DQ-07 | CRITICAL | CFO + CFI + CFF ≈ net cash flow |
| DQ-08 | WARNING | Implied tax rate within [0%, 60%] |
| DQ-09 | WARNING | Dividend percentage ≤ 25% |
| DQ-10 | WARNING | Document URL format validity |
| DQ-11 | WARNING | EPS sign consistent with net profit sign |
| DQ-12 | CRITICAL | Balance sheet liability components sum to total_liabilities |
| DQ-13 | CRITICAL | Year within valid range [1990, 2027] |
| DQ-14 | CRITICAL | Ticker format validity |
| DQ-15 | CRITICAL | Mandatory fields non-null |
| DQ-16 | WARNING | Company has ≥ 5 years of P&L history |

## Definition of Done — verified results

| Exit criterion | Target | Actual |
|---|---|---|
| Companies loaded | 92 | **92** ✅ |
| `PRAGMA foreign_key_check` violations | 0 | **0** ✅ |
| CRITICAL load rejections | 0 | **0** ✅ |
| CRITICAL DQ violations | 0 | **0** ✅ |
| Unit tests passing | 35+ | **102** ✅ |
| Manual review of flagged companies | 5 | **5** (see `docs/day06_manual_review.md`) ✅ |
| profitandloss rows | ~1,276 | **1,274** ✅ |
| balancesheet rows | ~1,312 | **1,352** ✅ |
| cashflow rows | ~1,187 | **1,205** ✅ |
| stock_prices rows | 5,520 | **5,520** ✅ |

Remaining WARNING-level findings (95 total, 0 CRITICAL) are exactly the
kind of issues a Day 06 manual review is meant to triage — a handful
of unbalanced balance sheets, OPM mismatches, negative sales rows,
EPS sign inconsistencies, malformed document URLs, and the 5 companies
with sparse year coverage documented in `docs/day06_manual_review.md`.

## Running individual pieces

```bash
# Regenerate the synthetic dataset only
.venv/bin/python src/etl/generate_synthetic_data.py

# Full load (drops and recreates nifty100.db from data/raw/*.xlsx)
.venv/bin/python -m src.etl.loader

# Compute financial_ratios from already-loaded data
.venv/bin/python src/etl/compute_ratios.py

# Run all 16 DQ rules and print a summary
.venv/bin/python -m src.etl.validator

# Run a single test file
.venv/bin/python -m pytest tests/etl/test_validator.py -v
```

## Configuration

All paths and DQ tolerances are configurable via `.env` (copy from
`.env.example`): `DB_PATH`, `RAW_DATA_DIR`, `OUTPUT_DIR`,
`MIN_YEAR_COVERAGE`, `BS_BALANCE_TOLERANCE`, `OPM_TOLERANCE`,
`NET_CASH_TOLERANCE`.

## Next steps (out of scope for Sprint 1)

The Makefile includes placeholder `dashboard` and `api` targets,
explicitly out of scope for this "Data Foundation" sprint — they're
reserved for whichever future sprint builds a reporting layer or
service API on top of this database.
