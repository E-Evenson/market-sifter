## Overview
market-sifter is an automated screening pipeline for publicly traded equities. 
It fetches and stores fundamental financial data, then applies systematic screening criteria based on value investing principles (Graham-influenced) to surface candidates across a set of configurable strategies (e.g. defensive, enterprising, quality, and growth).

The initial focus is NYSE equities via the FMP API free tier. TSX and additional exchanges are planned as the project matures.

The goal is to replace manual fundamental analysis with a repeatable, auditable pipeline that can be extended to new strategies and markets over time.

## Stack
- **Dagster** - Modern orchestration tool. Asset based model makes dependencies and lineage clear across pipeline stages, with excellent dbt integration using dagster-dbt
- **Python** - API ingestion. Keeping the ingestion layer thin, fetch and load only, no transformation
- **DuckDB** - Embedded columnar database, native JSON parsing, and excellent dbt integration. No infrastructure overhead, and well suited to the scale of fundamental financial data
- **dbt** - Modern tool for version controlled, templated SQL with dependency mapping and integrated testing. All transformation logic lives here, keeping strategy screening criteria auditable and easy to extend.
- **Poetry** — dependency management and packaging.

## Data Architecture
market-sifter follows an ELT pattern. Data is extracted and loaded before any transformations are done. This simplifies ingestion and retains raw data to allow for reprocessing if domain logic changes.

### Layers
- **Raw** - raw API responses saved as timestamped JSON files on disk. Loaded into raw DuckDB layer for querying.
- **Staging** - dbt models clean, type, and normalize raw data. No business logic applied. 
- **Intermediate** - dbt models apply domain logic/fundamentals calculations once and are reused by all strategies
- **Marts** - one model per investment strategy, filters applied against intermediate layer

## Repo Structure
market-sifter uses a src layout, with the Dagster project under `src/market_sifter/`. dbt lives alongside `src/` at the project root, as it is its own project

```
market-sifter/
├── dbt/
│   ├── dbt_project.yml
│   ├── profiles.yml
│   ├── models/
│   │   ├── staging/
│   │   ├── intermediate/
│   │   └── marts/
│   ├── macros/
│   └── tests/
├── src/
│   └── market_sifter/
│       ├── __init__.py
│       ├── definitions.py
│       └── defs/
│           ├── __init__.py
│           ├── ingestion/
│           │   └── fmp/
│           │       ├── assets.py
│           │       └── resources.py
│           └── transformation/
│               └── dbt/
│                   ├── assets.py
│                   ├── resources.py
│                   └── partitions.py
├── data/
│   └── raw/
├── docs/
├── tests/
├── dagster.yaml
├── workspace.yaml
└── pyproject.toml
```

## Data Sources
The Financial Modelling Prep (FMP) API is the primary data source. I am using the free tier, which has a limit of 250 requests per day, and 512MB bandwidth per rolling 30 days. To work within these constraints market-sifter will ingest high level data in bulk, and use that to prioritize which stocks to do deep dives on.

## Ingestion Design
Ingestion is done with a Python client fetching data from the FMP API. Raw responses are loaded into the DuckDB raw layer and saved as timestamped JSON files to disk. No transformation occurs at ingestion.

Ingestion follows a tiered approach to manage rate limits:
1. **Universe** — bulk high-level data for all stocks on the exchange
2. **Deep dive** — detailed financial statements for prioritized tickers, batched across days to stay within daily request limits

Raw JSON is retained on disk as an immutable record of what was fetched, allowing the DuckDB layers to be rebuilt from scratch if needed.

## Pipeline Design
The pipeline is orchestrated by Dagster, with each stage modelled as an asset

### Assets
- **universe** - fetches bulk high level data for all stocks on the exchange. Output drives prioritization for deep dives.
- **deep_dive** - fetches detailed financial statements for prioritized tickers, batched across days to stay within rate limits
- **dbt models** - staged, intermediate, and mart models run after ingestion, transforming raw data into screened strategy outputs

### Scheduling
- universe refreshes daily to follow price, market cap, new listings, de-listings, and prioritize tickers for deep dive
- earnings calendar checked daily to determine which tickers have new filings available
- deep dives run daily, processing batches of highest priority eligible tickers. Tickers already processed since their last filing are excluded
- deep dive batch size is TBD pending API request cost per deep dive
- dbt runs after universe refresh to update staging and intermediate layers, keeping priority queue current
- dbt runs after each deep dive batch to update strategy marts with newly ingested statements
- when no new filings are eligible, remaining daily requests are used to backfill historical statements for already-processed tickers

## Open Questions
- How many API calls does a deep dive take
- Exact details available in free tier
- Handling fiscal year variance across companies
- Prioritization criteria
- Strategy screening criteria
- Are price dependent ratios tracked historically or just instantaneously
- Price alert design

## Future Considerations
- Expansion to different markets
- Additional data sources
- Paid tier upgrade criteria
- Price alert when ratios cross thresholds
- Historical backfill of financial statements
- Backtesting strategy performance against historical data
