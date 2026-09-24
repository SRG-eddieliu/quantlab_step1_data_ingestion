# QuantLab Data Ingestion (Legacy Component)

> **Consolidated implementation:** start with [Systematic Alpha Lab](https://github.com/SRG-eddieliu/systematic-alpha-lab) and its [`data_pipeline` package](https://github.com/SRG-eddieliu/systematic-alpha-lab/tree/main/src/systematic_alpha_lab/data_pipeline).
>
> This repository is retained as an earlier standalone implementation. The notes below describe that version, including its original paths and assumptions. They are not evidence of a validated investment strategy. See the consolidated repository for current scope, evaluation limitations, and planned work.

## Original Component Documentation

Automated pipeline for ingesting, transforming, and serving financial and economic time-series data using WRDS constituents and Alpha Vantage MCP tools. Data is written to external Parquet stores (raw and final long format), with logging and quality checks.

## Layout
- Repository root: code and configs.
- External data root (sibling to repo): `../data/`
  - Raw landing zone: `../data/data-raw/`
  - Final analytical tables: `../data/data-processed/` (e.g., `price_daily.parquet`, `company_overview.parquet`, `FAMA_FRENCH_FACTORS.parquet`)

## Credentials
Create `credentials.yml` at the repo root (gitignored) with Alpha Vantage and WRDS credentials:
```yaml
alphavantage_api: "<free-key>"
alphavantage_api_paid: "<paid-key>"
wrds_username: "<wrds-username>"
wrds_password: "<wrds-password>"
```

## mapping notes
Constituents are pulled from `crsp_a_indexes.dsp500list_v2` using `permno`, `mbrstartdt`, `mbrenddt` and mapped to tickers via `crsp.msenames` date-overlap joins.

## Key functions
- `run_ingestion(date_start=None, date_end=None, ...)`: pulls constituents from WRDS, fetches Alpha Vantage via REST (MCP only for analytics if used), saves raw Parquet per ticker/endpoint. If dates not provided, uses `config/datalist.yml` defaults.
- `transform_raw_to_final()`: builds domain-specific final tables in `../data/data-processed/`: `price_daily.parquet`, `price_weekly.parquet`, `economic_indicators.parquet`, `company_overview.parquet`, `FAMA_FRENCH_FACTORS.parquet` (if fetched), and fundamentals split by statement. Fundamentals now also emit separate quarterly/annual files (e.g., `fundamentals_earnings.parquet` plus `fundamentals_earnings_quarterly.parquet` and `fundamentals_earnings_annual.parquet`; same for income_statement, balance_sheet, cash_flow, earnings_estimates, dividends, splits).
- `run_quality_checks(dataset="price_daily")`: basic completeness/consistency/bounds checks on price datasets.
- `get_final_data(dataset="price_daily", tickers=None, start_date=None, end_date=None)`: read and filter a chosen final dataset.
- `export_fundamental_failures()`: scans `fundamentals.parquet` for “invalid api call” rows and writes a CSV with suggested API calls to re-run (default: `data/data-processed/failures_temp.csv`).
- `export_company_overview_failures()`: scans `company_overview.parquet` for “invalid api call” rows and writes a CSV with suggested API calls to re-run (default: `data/data-processed/company_overview_failures.csv`).
- `export_all_failures()`: scans all raw Parquets for “invalid api call” or rate-limit payloads and writes a single CSV (`data/data-processed/failures_all.csv`) with ticker/function/API URL to rerun.
- `refetch_failures(failures_csv)`: re-fetch failed fundamentals/company overview calls listed in a CSV (ticker/function) and overwrite raw Parquets; rerun `transform_raw_to_final()` afterward.
- `run_ingestion(..., fetch_ff=True)`: optionally pulls Fama-French factors from WRDS (`ff_all.factors_daily`) and writes `data/data-processed/FAMA_FRENCH_FACTORS.parquet`.
- Notebook: [`notebooks/pipeline_demo.ipynb`](notebooks/pipeline_demo.ipynb) shows end-to-end usage (ingestion, transform, quality checks, failure handling, FF factors-only fetch).

## API usage notes
- Time series (`TIME_SERIES_DAILY_ADJUSTED`, `TIME_SERIES_WEEKLY_ADJUSTED`) are fetched via the Alpha Vantage REST API with `datatype=csv` and `outputsize=full` for full history.
- Fundamentals, listings/calendars, and economic endpoints use the REST API with their function names (e.g., `OVERVIEW`, `INCOME_STATEMENT`, `LISTING_STATUS`, etc.)
- MCP is used for tool discovery
- Ingestion writes a unique ticker list (`wrds_sp500_unique_tickers.parquet`) and uses it to drive API calls; the daily constituent Parquet is still written for auditing.
- Alpha Vantage source:https://www.alphavantage.co/documentation/
- Ingestion has a `resume` flag (default True): it skips endpoints where a non-empty Parquet already exists, so you can rerun after timeouts without re-fetching everything. 
      run_ingestion(date_start=date(2024, 1, 2), date_end=date(2025, 1, 2), sleep_seconds=12.0, use_paid_key=True, resume = False)

## MCP configuration
The repo includes `.vscode/mcp.json` pointing to the Alpha Vantage MCP HTTP endpoint. Ensure the API key in that file matches your credentials.yml.

## Considerations
- WRDS crsp_a_indexes.dsp500list_v2 (annual) used to create SP500 constituents annual list. Daily constituents were then expanded based on annual information which introduce look-ahead/survivorship bias. With WRDS daily access these bias can be removed
