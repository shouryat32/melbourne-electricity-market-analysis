# Table Schema Reference

Quick reference for every table in the pipeline — what it contains, key columns, and when to use it.

**Database:** `energy_analytics` | **Platform:** Azure Databricks (Delta Lake)

---

## Which table should I use?

| What you're trying to do | Table |
|--------------------------|-------|
| General analysis or dashboards | `silver_energy_melbourne_extended` |
| Hourly patterns across 3 years | `gold_hourly_summary` |
| Day-by-day trends | `gold_daily_summary` |
| Consumer or solar owner advice | `gold_consumer_insights` |
| Price spikes and negative events | `gold_price_insights` |
| Seasonal patterns | `gold_seasonal_analysis` |
| Variable correlations | `gold_correlation_matrix` |
| ML predictions | `gold_ml_predictions` |
| Raw 5-minute prices | `bronze_aemo_extended_prices` |
| Generation by fuel type | `bronze_generation_extended` |
| Raw weather | `bronze_weather_extended_melbourne` |

---

## Bronze Layer — Raw Data

### `bronze_aemo_extended_prices`
5-minute wholesale prices and demand from AEMO. 313,344 records.

| Column | Type | Notes |
|--------|------|-------|
| `SETTLEMENTDATE` | string | Settlement datetime |
| `REGION` | string | Market region (VIC1) |
| `RRP` | double | Wholesale price $/MWh — can be negative or spike >$1,000 |
| `TOTALDEMAND` | double | Total demand (MW) |
| `PERIODTYPE` | string | Period type |

---

### `bronze_generation_extended`
Hourly generation by fuel type — one row per fuel per hour. 196,368 records.

| Column | Type | Notes |
|--------|------|-------|
| `timestamp` | timestamp | Hour |
| `fueltech_group` | string | coal, gas, hydro, wind, solar, battery |
| `energy_mwh` | double | Generation (MWh) — negative for battery charging |
| `network_region` | string | VIC1 |

---

### `bronze_generation_hourly`
Same as above but aggregated — one row per hour with all fuels combined. 26,126 records.

| Column | Type | Notes |
|--------|------|-------|
| `hour` | timestamp | Hour |
| `scheduled_generation` | double | Coal + gas + hydro (MWh) |
| `semischeduled_generation` | double | Wind + solar (MWh) |
| `coal` / `gas` / `hydro` / `wind` / `solar` | double | Individual fuels (MWh) |
| `renewable_pct` | double | (wind + solar + hydro) / total × 100 |

---

### `bronze_weather_extended_melbourne`
Hourly weather from Open-Meteo for Melbourne. 26,376 records.

| Column | Type | Notes |
|--------|------|-------|
| `datetime` | timestamp | Hour |
| `temperature` | double | °C |
| `humidity` | long | % |
| `wind_speed` | double | km/h |
| `solar_radiation` | double | W/m² |
| `cloud_cover` | long | % |

---

### `bronze_aemo_live_prices` / `bronze_aemo_generation` / `bronze_bom_weather`
Live data appended every hour by `phase1_setup`. Same structure as the historical equivalents above, partitioned by `settlement_date` and `region`.

---

## Silver Layer — Analytics-Ready

### `silver_energy_melbourne_extended` ⭐
The primary table. All three bronze sources joined on datetime, cleaned to hourly resolution, with derived features added. 26,156 records.

| Column | Type | Notes |
|--------|------|-------|
| `datetime_aest` | timestamp | Primary time key |
| `price` | double | Avg hourly wholesale price $/MWh |
| `price_tier` | string | negative / cheap / normal / expensive / spike |
| `demand_mw` | double | Avg demand (MW) |
| `renewable_pct` | double | % of generation from renewables |
| `coal` / `gas` / `hydro` / `wind` / `solar` | double | Generation by fuel (MWh) |
| `avg_temp` | double | Temperature °C |
| `avg_wind_speed` | double | Wind speed km/h |
| `solar_radiation` | double | W/m² |
| `hour_of_day` | integer | 0–23 |
| `day_of_week` | integer | 1–7 |
| `month` | integer | 1–12 |
| `is_peak` | double | 1 if 5pm–9pm, else 0 |

**Price tiers:** negative (<$0) · cheap ($0–50) · normal ($50–100) · expensive ($100–300) · spike (>$300)

---

## Gold Layer — Business Insights

### `gold_hourly_summary`
3-year averages by hour of day. 24 records — one per hour.

Key columns: `hour_of_day`, `avg_price`, `avg_renewable_pct`, `negative_price_count`, `spike_count`, `is_peak_hour`

---

### `gold_daily_summary`
Daily stats across all 1,088 days in the dataset.

Key columns: `settlement_date`, `avg_price`, `peak_demand_mw`, `avg_renewable_pct`, `negative_price_hours`, `spike_hours`, `savings_opportunity_score`

---

### `gold_consumer_insights`
Consumer and solar owner advice by hour of day. 24 records.

Key columns: `hour_of_day`, `wholesale_price_mwh`, `tou_likely_rate`, `estimated_retail_cost_kwh`, `consumer_advice`, `solar_owner_advice`

---

### `gold_price_insights`
Every negative pricing and spike event across 3 years. 6,313 records.

Key columns: `datetime_aest`, `price`, `event_type` (negative_pricing / price_spike), `renewable_pct`, `demand_mw`, `avg_temp`

---

### `gold_seasonal_analysis`
Season × temperature bucket breakdown. ~100 records.

Key columns: `season`, `temp_bucket`, `avg_renewable_pct`, `avg_price`, `avg_demand`

---

### `gold_correlation_matrix`
Pearson correlations between key variables. ~10 records.

| Pair | Correlation | Strength |
|------|-------------|----------|
| Demand vs Price | +0.589 | Strong |
| Temperature vs Demand | -0.444 | Moderate |
| Temperature vs Price | -0.379 | Moderate |
| Renewable % vs Price | -0.160 | Weak |

---

### `gold_ml_predictions`
All 26,156 historical rows scored by the three ML models.

Key columns: `datetime_aest`, `actual_price`, `predicted_price`, `actual_demand`, `predicted_demand`, `spike_probability`

---

*For notebook documentation see [README.md](../notebooks/README.md)*
