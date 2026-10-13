# UK Carbon Intensity Pipeline

## Business question
Which times of day — and which days of the week — have the lowest carbon
intensity on the UK electricity grid? In other words: when is the
"greenest" time to use power?

## Architecture
Public API → Bronze (raw) → Silver (cleaned) → Gold (aggregated) → Power BI report

![Architecture diagram](architecture-diagram.png)

- **Source:** National Grid ESO Carbon Intensity API
  (https://api.carbonintensity.org.uk) — free, public, no authentication required
- **Platform:** Databricks Free Edition, Unity Catalog, PySpark
- **Orchestration:** Databricks Jobs — Bronze runs on a fixed schedule,
  Silver and Gold trigger automatically on upstream table updates
- **Version control:** This repo, synced via Databricks Repos (Git integration)
- **Reporting:** Power BI, connected to Databricks SQL Warehouse

## How it works

### Bronze — `01_bronze_ingestion`
Calls the Carbon Intensity API and appends the raw, unparsed response into
`carbon_intensity.bronze.intensity_raw`, tagged with the time it was pulled.
Nothing is cleaned or interpreted here — Bronze preserves exactly what the
API returned, so the pipeline can be reprocessed from scratch if the
downstream parsing logic ever needs to change.

### Silver — `02_silver_cleaning`
Reads all of Bronze, parses the nested JSON into flat columns (period start,
forecast, actual, intensity band), removes duplicate time periods, and
derives `hour_of_day` and `day_of_week` for analysis. Writes to
`carbon_intensity.silver.intensity_clean`. Triggers automatically whenever
Bronze is updated.

### Gold — `03_gold_processing`
Aggregates Silver into three tables that directly answer the business
question:
- `avg_by_hour` — average intensity for each hour of the day
- `avg_by_hour_dow` — average intensity by hour, split by day of week
- `greenest_hours` — the top 3 lowest-intensity hours, ranked

Triggers automatically whenever Silver is updated.

### Dashboard — Power BI
A single bar chart showing average carbon intensity for each hour of the
day (0–23), with the lowest-intensity hour highlighted in green. Built from
`gold.avg_by_hour`, connected live via Databricks SQL Warehouse.

## Finding: 
Carbon intensity is consistently lowest around midday, with
hour 12 averaging roughly 83 gCO2/kWh — the lowest point in the daily cycle.

## How to reproduce
1. Sign up for Databricks Free Edition
2. Create a Unity Catalog catalog `carbon_intensity` with schemas `bronze`,
   `silver`, `gold`
3. Clone this repo into Databricks via Workspace → Repos
4. Run `01_bronze_ingestion` once manually, then schedule it (every 30 min)
   via Jobs & Pipelines
5. Set `02_silver_cleaning` and `03_gold_processing` to trigger on upstream
   table updates
6. Connect Power BI Desktop to your Databricks SQL Warehouse (Get Data →
   Databricks) and build visuals from the `gold` schema tables

## Key decisions
- **Used `coalesce(actual, forecast)`** in Silver — the API's `actual`
  field is null until a period settles, so this guarantees one usable
  number at all times.
- **Bronze stores the full raw API response as text, unparsed** — so
  Silver's parsing logic can be changed and re-run against full history
  without re-calling the API.
- **Event-based triggers for Silver and Gold**, rather than fixed
  schedules — each stage only re-runs when its actual input data changes,
  rather than on a blind timer.
- I chose Databricks because Fabric’s free trial wasn’t available to new accounts when I started this project, and it gave me a chance to learn a platform outside my current Fabric-based work experience.

## Limitations / what I'd improve with more time
- Deduplication in Silver uses `dropDuplicates`, which keeps an arbitrary
  copy of a repeated time period rather than specifically the most recently
  ingested one — fine at this scale, but I'd use a ranked window function
  for a production version.
- No automated data-quality monitoring beyond the row-count and null checks
  printed in each notebook.
- Currently only uses the `/intensity` endpoint; the API's `/generation`
  endpoint (fuel mix — wind, solar, gas, etc.) would let the project explain
  *why* certain hours are greener, not just *which* ones are.
- Built avg_by_hour_dow and greenest_hours as Gold tables but haven’t yet visualized them — the current report only           surfaces the overall hourly pattern, not day-of-week variation

## A note on how this was built
I built this while learning Databricks, PySpark, and medallion architecture
from a BI/analytics background, using AI assistance to learn concepts and
debug issues along the way. I made the design decisions, debugged the real
errors that came up (schema mismatches, timestamp parsing, duplicate
ingestion), and can explain the logic and trade-offs behind every layer of
this pipeline.
