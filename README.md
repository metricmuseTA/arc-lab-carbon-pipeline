# ARC LAB · Grid Carbon Intensity Pipeline
ARC LAB · Grid carbon intensity pipeline · WattTime CAISO_NORTH → Databricks Bronze/Silver/Gold → AI scheduling optimization


> **Grid carbon intensity in CAISO_NORTH varies by 17× between the cleanest and dirtiest hours of the day. This pipeline quantifies that gap and calculates the avoidable carbon cost of AI workloads scheduled at peak vs off-peak windows.**

Built by [Tiffani Anderson](https://github.com/metricmuseTA) · Metric Muse LLC · ARC LAB (AI Renewable Consumption Laboratory)

---

## The Finding

| Metric | Value |
|---|---|
| Peak intensity (dirtiest hour avg) | **1,030 lbs CO₂/MWh** |
| Off-peak intensity (cleanest hour avg) | **58.5 lbs CO₂/MWh** |
| Grid variability ratio | **17.6×** |
| Intervals sampled | 2,016 |
| Data source | WattTime CAISO_NORTH |

**An AI training job scheduled at peak hours carries over 17× the carbon cost of the same job run at the optimal off-peak window.** This is not a rounding error — it is a scheduling decision with real, measurable consequences.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        ARC LAB PIPELINE                     │
│                                                             │
│   WattTime API          Databricks         AI/BI Dashboard  │
│   (CAISO_NORTH)                                             │
│                                                             │
│   /v3/historical  ──►  BRONZE        ──►                   │
│   lbs_co2_per_mwh      Raw ingest         4-panel dashboard │
│                         8,760 rows    ──►  + Genie NL layer │
│                              │                              │
│                         SILVER                              │
│                         Cleaned + typed                     │
│                         8,612 rows                          │
│                              │                              │
│                         GOLD                                │
│                         carbon_intensity_hourly             │
│                         8,580 rows                          │
│                         + peak_flag                         │
│                         + ai_carbon_multiplier              │
└─────────────────────────────────────────────────────────────┘
```

**Medallion layers:**

- **Bronze** — Raw WattTime API response written to Delta Lake, append-only, no transforms
- **Silver** — Type enforcement, null removal (148 rows), dedup on `(timestamp, ba)` composite key
- **Gold** — Hourly aggregates, `peak_flag` boolean, `ai_carbon_multiplier` float, optimized for Genie NL queries

---

## Gold Table: Data Dictionary

**Table:** `bootcamp_students.gold.carbon_intensity_hourly`

| Column | Type | Description |
|---|---|---|
| `hour_ts` | TIMESTAMP_NTZ | Hour-truncated UTC timestamp |
| `ba` | STRING | Balancing authority — CAISO_NORTH |
| `avg_intensity` | DOUBLE | Average lbs CO₂/MWh for that hour |
| `min_intensity` | DOUBLE | Minimum lbs CO₂/MWh for that hour |
| `max_intensity` | DOUBLE | Maximum lbs CO₂/MWh for that hour |
| `record_count` | LONG | Number of 5-min intervals aggregated |
| `peak_flag` | BOOLEAN | TRUE when avg_intensity > 700 lbs/MWh |
| `ai_carbon_multiplier` | DOUBLE | Ratio of this hour's intensity vs cleanest hour avg |
| `hour_of_day` | INT | 0–23, extracted for dashboard grouping |
| `day_type` | STRING | 'Weekday' or 'Weekend' |
| `recommendation` | STRING | 'Optimal' / 'Moderate' / 'Avoid' scheduling label |

---

## Dashboard

The AI/BI Dashboard (`ARC LAB — Grid Carbon Intensity`) is live in the Databricks workspace and includes:

- **KPI tiles** — Avg Carbon Intensity, Cleanest Hour Avg, Dirtiest Hour Avg, Intervals Sampled
- **CO₂ Intensity by Hour of Day** — weekday vs weekend line chart
- **Scheduling Opportunity by Hour** — bar chart color-coded Optimal / Moderate / Avoid
- **CO₂ Range by Hour (Weekday)** — min/max/avg band chart
- **Weekday CO₂ Savings & Scheduling** — table with lbs CO₂ saved vs peak per hour

Natural language queries are available via the **Genie** interface — ask questions like:
> *"What is the best hour to run an AI workload on a weekday?"*
> *"Which hours should I avoid for carbon-intensive compute?"*

---

## Notebook Execution Order

```
00_setup/
  01_create_schemas.py       ← Run once
  02_create_tables.py        ← Run once

01_ingest/
  01_watttime_auth.py        ← Authenticates, stores token
  02_bronze_ingest.py        ← Fetches historical data → Bronze
  03_backfill_historical.py  ← Backfill for gap coverage

02_transform/
  01_bronze_to_silver.py     ← Cleans + types → Silver
  02_silver_to_gold.py       ← Aggregates + flags → Gold

03_quality/
  01_dq_checks.py            ← 18 DQ checks, all PASS

04_dashboard/
  (Configured in Databricks AI/BI UI)
```

---

## Setup & Configuration

### Requirements
- Databricks workspace with Unity Catalog
- WattTime API account (free tier covers CAISO_NORTH)
- Python 3.11+

### Secrets
This pipeline uses Databricks secrets — no credentials are stored in notebooks.

```python
# Create secret scope
databricks secrets create-scope arc-lab

# Store WattTime credentials
databricks secrets put-secret arc-lab watttime-username
databricks secrets put-secret arc-lab watttime-password
```

Access in notebooks:
```python
from databricks.sdk import WorkspaceClient
w = WorkspaceClient()
username = w.secrets.get_secret(scope="arc-lab", key="watttime-username")
```

### Known Workarounds (Databricks Bootcamp Environment)

These are documented for reproducibility — some are environment-specific:

```python
# TIMESTAMP_NTZ epoch math — direct cast fails silently
unix_timestamp(col("ts_utc").cast("timestamp"))  # ✅
col("ts_utc").cast("long")                        # ❌ returns null

# LangChain import path
from databricks_langchain import ...              # ✅
from langchain_databricks import ...              # ❌ deprecated

# No INTERVAL syntax in Spark SQL — use epoch math
ts_utc >= current_timestamp() - (7 * 86400)      # ✅
ts_utc >= current_timestamp() - INTERVAL 7 DAYS  # ❌
```

---

## About ARC LAB

**ARC LAB** (AI Renewable Consumption Laboratory) is an independent research initiative building the first open, publicly accessible dataset measuring AI workload energy consumption and its carbon cost.

**The three-layer research architecture:**

1. **Grid Carbon Intensity** — real-time and historical carbon intensity by grid region (this repo)
2. **AI Energy Consumption** — published research RAG pipeline quantifying energy draw of major AI training and inference workloads
3. **Intersection** — joins grid signal with AI workload timing to compute avoidable carbon cost per workload class, per region

The core thesis: AI's energy footprint is growing faster than our ability to measure it. ARC LAB is building the open infrastructure to change that.

**Stack:** Databricks · Delta Lake · WattTime API · Python · Spark SQL · dbt Core · LangChain · Vector Search

---

## License

MIT License — see [LICENSE](LICENSE)

---

*Built during the DataExpert.io Databricks Bootcamp, March 2026*
*Tiffani Anderson · [Metric Muse LLC](https://github.com/metricmuseTA) · La Mesa, CA*
