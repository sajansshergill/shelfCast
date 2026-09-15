# ShelfCast — Grocery Demand Forecasting Platform

An end-to-end data platform that ingests SKU-level retail sales, models raw data into a
forecast-ready layer, generates demand forecasts on a schedule, and serves them to both an
application backend and an LLM. Built to mirror the data-engineering problems that grocery
forecasting companies solve in production: onboarding messy customer datasets, running
incremental pipelines many times a day, and turning billions of rows into decisions store
operators can act on.

> **TL;DR** — M5 retail data → Dagster asset graph → LightGBM forecast → FastAPI + MCP serving.
> Config-driven onboarding for new datasets. Incremental, partitioned runs. Dockerized, with a
> Terraform stub for GCP.

---

## Why this exists

Grocery demand forecasting is a data-engineering problem before it is a modeling problem. The
hard parts are: normalizing every retailer's differently-shaped data into one canonical schema,
running the pipeline reliably and incrementally at scale, and exposing the output where people
(and agents) can use it. ShelfCast is a working, honest miniature of that system — the model is
deliberately standard so the pipeline discipline can be the star.

## What it does

- **Ingests** SKU × store × day sales, inventory, and calendar/price data into a warehouse layer.
- **Onboards new datasets by config** — a YAML schema mapper normalizes an unfamiliar column
  layout into the canonical schema, so a "new chain" is a mapping file, not a code change.
- **Builds features and forecasts** — lag and rolling-window features feed a LightGBM model,
  evaluated with a real held-out backtest.
- **Runs incrementally** — Dagster partitions and schedules update only what changed, simulating
  multiple daily runs rather than full refreshes.
- **Serves forecasts** — a FastAPI service exposes forecast and reorder endpoints, and an MCP
  server wraps the same logic as tools so an LLM can answer "how much of product X should
  store Y order tomorrow?"

## Architecture

```
                    ┌─────────────────────────────────────────────┐
   raw retail data  │                 DAGSTER                       │
   (M5 / customer)  │   asset graph, partitioned + scheduled        │
        │           │                                               │
        ▼           │   raw ──► staged ──► features ──► forecast ──► serving
   ┌─────────┐      │    ▲          ▲          ▲           ▲          │
   │ schema  │──────┼────┘   (canonical schema)      (LightGBM)      │
   │ mapper  │ YAML │                    Pandas / Dask                │
   └─────────┘      └───────────────────────┬───────────────────────┘
                                             │
                          ┌──────────────────┴──────────────────┐
                          ▼                                      ▼
                 ┌─────────────────┐                   ┌──────────────────┐
                 │  FastAPI service│                   │   MCP server     │
                 │  /forecast      │                   │  get_forecast()  │
                 │  /reorder       │                   │  suggest_reorder │
                 └─────────────────┘                   └──────────────────┘
                          │                                      │
                     app / dashboards                       LLM / agent
```

The warehouse layer runs on **BigQuery** (free tier) or **Postgres/DuckDB** locally, so the
whole thing costs nothing to run.

## Tech stack

| Layer            | Tools                                             |
|------------------|---------------------------------------------------|
| Orchestration    | Dagster (assets, partitions, schedules, sensors)  |
| Processing       | Python, Pandas, Dask                              |
| Storage          | BigQuery / Postgres / DuckDB                       |
| Modeling         | LightGBM, scikit-learn                            |
| Serving          | FastAPI, MCP (Anthropic tool-use)                 |
| Infra            | Docker, Terraform (GCP stub)                       |

## Dataset

[M5 Forecasting — Accuracy](https://www.kaggle.com/competitions/m5-forecasting-accuracy)
(Walmart): ~30,000 product-store series of daily unit sales, hierarchical by item / department /
store / state, with calendar and price data. Real grocery demand, and large enough (tens of
millions of rows when unpivoted) to exercise incremental and distributed processing honestly.

## Getting started

### Prerequisites

- Python 3.11+
- Docker (optional, for the containerized run)
- A Kaggle account to download the M5 dataset

### Setup

```bash
git clone https://github.com/sajansshergill/shelfcast.git
cd shelfcast

python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

# place the M5 CSVs in data/raw/ (see data/README.md)
```

### Run the pipeline

```bash
# launch the Dagster UI
dagster dev

# or materialize the full asset graph headless
dagster asset materialize --select '*'
```

### Serve forecasts

```bash
# REST API
uvicorn shelfcast.api.main:app --reload
# → http://localhost:8000/docs

# MCP server (for use with Claude Desktop / an agent)
python -m shelfcast.mcp.server
```

### With Docker

```bash
docker compose up
```

## Onboarding a new dataset

Point the schema mapper at a new file's columns and map them to the canonical schema — no
pipeline code changes required:

```yaml
# configs/customers/new_chain.yaml
source: new_chain
mapping:
  store_id:   "Location_Nbr"
  sku_id:     "Item_Code"
  date:       "Txn_Date"
  units_sold: "Qty"
  price:      "Unit_Price"
date_format: "%m/%d/%Y"
```

The `raw → staged` asset reads this config and emits data in the canonical schema the rest of
the graph expects.

## Forecasting approach

- **Features:** sales lags (7/14/28-day), rolling means and std, calendar events, price change
  flags, and hierarchy encodings.
- **Model:** LightGBM regressor — the standard, strong M5 baseline. The point is a clean,
  reproducible pipeline, not a novel model.
- **Evaluation:** time-based backtest on a held-out horizon, reported as RMSE / MAPE (and
  WRMSSE if you want to match the competition metric). Metrics land in a `forecast_eval` asset
  so accuracy is tracked, not assumed.

## Roadmap

- [ ] Pub/Sub trigger (emulator) to kick incremental runs on new-data arrival
- [ ] Data quality checks as Dagster asset checks (nulls, ranges, freshness)
- [ ] Backfill + reconciliation for late-arriving data
- [ ] Model registry and per-run metric comparison
- [ ] Reorder-recommendation logic layered on top of raw forecasts

## Project layout

```
shelfcast/
├── configs/customers/     # per-dataset schema mappings (YAML)
├── shelfcast/
│   ├── assets/            # Dagster assets: raw, staged, features, forecast, serving
│   ├── mapping/           # schema mapper
│   ├── modeling/          # feature engineering + LightGBM
│   ├── api/               # FastAPI service
│   └── mcp/               # MCP server exposing forecast tools
├── infra/terraform/       # GCP resource stubs
├── data/                  # raw / interim / processed (gitignored)
├── docker-compose.yml
└── requirements.txt
```

## Author

**Sajan Shergill** — Data / AI Engineer
[Portfolio](https://sajansshergill.github.io) · [LinkedIn](https://linkedin.com/in/sajanshergill) · [GitHub](https://github.com/sajansshergill)

## License

MIT
