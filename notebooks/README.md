# Notebook Documentation

Five notebooks make up the full pipeline — one for live data collection, one for historical backfill, one for the silver layer, one for gold aggregations, and one for machine learning. They are designed to be run in the order below.

**Platform:** Azure Databricks | **Storage:** Delta Lake | **Language:** PySpark + Python

---

## Pipeline Overview

```
phase1_setup         ← runs every hour via Databricks scheduler (live data)
historical_data      ← one-time backfill (3 years)
        ↓
Energy_silver_layer  ← runs at 2am daily (incremental append)
        ↓
Energy_gold_layer    ← aggregations & insights
        ↓
ml_model             ← price, demand & spike prediction
```

---

## 01 — phase1_setup (Live Data Collection)

**Schedule:** Every hour via Databricks scheduler  
**Mode widget:** `collect` | `historical` | `all`

Collects live data from two sources and appends it to the bronze layer. This is the heartbeat of the pipeline — every run captures the latest market snapshot and Melbourne weather readings.

**What it does:**
Hits the AEMO live summary API to pull current wholesale prices and generation figures for VIC1, then pulls live weather observations from two BOM stations (Melbourne Airport and Laverton). Each record is hashed for deduplication and timestamped in AEST before being appended to its respective Delta table.

The `mode` widget controls behaviour — `collect` runs live ingestion only, `historical` skips it, and `all` runs everything.

**Inputs:** AEMO Live API, BOM Weather API (no authentication required for either)

**Outputs:**

| Table | Description |
|-------|-------------|
| `bronze_aemo_live_prices` | Live wholesale price + regulation prices per region |
| `bronze_aemo_generation` | Live scheduled & semi-scheduled generation per region |
| `bronze_bom_weather` | Temperature, humidity, wind speed from 2 Melbourne stations |

---

## 02 — historical_data (One-Time Backfill)

**Schedule:** Run once  
**Runtime:** ~2 hours

Downloads 3 years of historical data from three external APIs to provide the foundation the silver and gold layers are built on.

**What it does:**
Pulls 36 months of AEMO 5-minute price and demand CSV archives, fetches hourly weather data from Open-Meteo for Melbourne, and retrieves hourly generation by fuel type from the OpenElectricity API using 30-day chunks to stay within rate limits. After ingestion, raw generation data (one row per fuel type per hour) is aggregated into a single hourly row with scheduled, semi-scheduled, and individual fuel totals calculated.

**Inputs:** AEMO historical API, Open-Meteo archive, OpenElectricity API v4 *(API key required)*

**Outputs:**

| Table | Records | Description |
|-------|---------|-------------|
| `bronze_aemo_extended_prices` | 313,344 | 5-minute prices & demand |
| `bronze_weather_extended_melbourne` | 26,376 | Hourly weather observations |
| `bronze_generation_extended` | 196,368 | Hourly generation by fuel type |
| `bronze_generation_hourly` | 26,126 | Aggregated hourly generation totals + renewable % |

---

## 03 — Energy_silver_layer (Incremental Nightly Build)

**Schedule:** 2am daily via Databricks scheduler  
**Runtime:** ~15 minutes

Picks up any new bronze data since the last run and appends it to the silver tables. Each run checks the last processed timestamp and only processes new records — the tables grow incrementally rather than being rebuilt from scratch each night.

**What it does:**
Combines historical and live bronze price tables, deduplicates, and cleans to hourly resolution. Does the same for generation and weather. Adds price tier classifications (`negative`, `cheap`, `normal`, `expensive`, `spike`) and a peak hour flag (5pm–9pm). Then joins all three into the unified Melbourne silver table.

**Inputs:** All bronze tables (both historical and live)

**Outputs:**

| Table | Description |
|-------|-------------|
| `silver_prices` | Cleaned hourly prices (historical + live combined) |
| `silver_generation` | Cleaned hourly generation |
| `silver_weather` | Cleaned hourly weather |
| `silver_energy_melbourne` | Unified VIC1 table — prices, generation, weather joined |
| `silver_energy_melbourne_extended` | Full table with derived features (price tier, peak flag, time features) |

---

## 04 — Energy_gold_layer (Aggregations & Insights)

**Schedule:** After silver layer completes  
**Runtime:** ~5 minutes

Reads from the silver table and pre-calculates the aggregations that power the Power BI dashboard. Also runs the correlation and seasonal analyses that surface the project's key findings.

**What it does:**
Builds hourly and daily summary tables, extracts all extreme price events (spikes and negative pricing), generates consumer and solar owner advice by hour, and runs the seasonal × temperature analysis. Computes Pearson correlations between temperature, demand, renewable %, and price.

**Inputs:** `silver_energy_melbourne_extended`

**Outputs:**

| Table | Records | Description |
|-------|---------|-------------|
| `gold_hourly_summary` | 24 | Price, demand, renewable % averaged by hour of day |
| `gold_daily_summary` | 1,088 | Daily stats including savings opportunity score |
| `gold_consumer_insights` | 24 | Consumer and solar owner advice by hour |
| `gold_price_insights` | 6,313 | All negative pricing and spike events |
| `gold_seasonal_analysis` | ~100 | Season × temperature bucket breakdown |
| `gold_correlation_matrix` | ~10 | Pearson correlations between key variables |

**Key findings surfaced here:**
- Noon is the cheapest hour — $1.40/MWh average, 622 negative pricing events
- 6pm is the most expensive — $177.76/MWh average
- Summer paradox confirmed: hot days (>25°C) average **29% renewable** vs **55%** on cool days — high-pressure systems suppress wind and solar gains don't compensate
- Strongest price predictor: Demand vs Price correlation of **+0.589**

---

## 05 — ml_model (Price, Demand & Spike Prediction)

**Schedule:** Run after gold layer / on demand  
**Runtime:** ~30–60 minutes (training)

Trains three separate ML models on the full 26,156-hour dataset and scores all historical rows. Results are saved to a gold predictions table consumed by the AI Price Intelligence page of the dashboard.

**What it does:**
Builds cyclical time features (sin/cos encoding for hour, day of week, month) and lag features before training. Uses an 80/20 chronological train/test split. The price model applies a log transformation to handle the skewed distribution caused by spikes. The spike classifier uses class-weight balancing to handle the rarity of spike events (~1% of hours).

**Models:**

| Model | Algorithm | Target | Result |
|-------|-----------|--------|--------|
| Price prediction | XGBoost | $/MWh | R² = 0.89, MAE = $6.69/MWh |
| Demand forecasting | CatBoost | MW | R² = 0.98, MAPE = 2.5% |
| Spike classifier | XGBoost + Platt scaling | Spike / No spike | ROC-AUC = 0.99, Precision = 79% @ 0.7 threshold |

**Inputs:** `silver_energy_melbourne_extended`

**Outputs:**

| Table | Description |
|-------|-------------|
| `gold_ml_predictions` | All 26,156 rows scored with predicted price, demand, and spike probability |

---

