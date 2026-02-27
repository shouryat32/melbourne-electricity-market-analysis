# melbourne-electricity-market-analysis

> *Started with one question: is solar actually worth it in Melbourne?*

After hearing the same debate over and over, I decided to stop speculating and look at the data. The answer was clear — every Melbourne zone rates **A-tier** for solar viability against international benchmarks, with an average payback period of around **6 years**. But that answer opened a much bigger question: what's actually going on inside Victoria's electricity market? This project is that deeper investigation.

**3 years of Victorian grid data. 26,156 hours. A full medallion pipeline on Azure Databricks. Three ML models. One Power BI dashboard.**

---

## Key Findings

- 🕛 **Cheapest hour:** 12pm — avg wholesale price of $1.40/MWh
- 🕕 **Most expensive hour:** 6pm — avg $177.76/MWh (a $176/MWh spread)
- ⚡ **Negative pricing** (grid oversupply) occurred in **23% of all hours** over 3 years
- 🌡️ **Summer paradox** — on Victoria's hottest days, renewable generation actually *falls*. High-pressure systems kill wind speed, and solar gains don't compensate. Hot days (>25°C) average just **29% renewable** vs **55%** on cool days
- 🤖 **ML price model** achieves R² = 0.89 | **Demand model** R² = 0.98 | **Spike classifier** ROC-AUC = 0.99

---

## Tech Stack

| Layer | Technology |
|-------|------------|
| Data platform | Azure Databricks + Delta Lake |
| Processing | PySpark |
| ML models | XGBoost, CatBoost |
| Model serving | FastAPI on Render |
| Visualisation | Power BI Desktop |
| Data sources | AEMO, OpenElectricity API v4, Open-Meteo |

---

## Repository Structure

```
melbourne-electricity-market-analysis/
│
├── notebooks/
│   ├── 01_historical_data_download.ipynb      # Bronze — 3yr data ingestion
│   ├── 02_bronze_generation_aggregation.ipynb  # Bronze — aggregate to hourly
│   ├── 03_silver_layer_build.ipynb             # Silver — join & transform
│   ├── 04_gold_layer_analytics.ipynb           # Gold — business insights
│   └── 05_bronze_validation.ipynb              # Data quality checks
│
├── models/
│   ├── price_model_xgboost/
│   ├── demand_model_catboost/
│   └── spike_classifier_xgboost/
│
├── api/
│   └── main.py                                 # FastAPI serving layer
│
├── docs/
│   ├── NOTEBOOKS_DOCUMENTATION.md
│   └── TABLE_SCHEMA_REFERENCE.md
│
└── README.md
```

---

## Dashboard

The Power BI dashboard covers four pages — Price Intelligence, Generation & Renewables, Weather Correlation, and AI Price Intelligence. It is available on request as it is not publicly hosted.

> 📩 Reach out via GitHub or LinkedIn if you'd like a walkthrough.

---

## Data Coverage

**Region:** Victoria, Australia (VIC1)  
**Period:** March 2023 – February 2026  
**Records:** 26,156 complete hourly observations  
**Sources:** AEMO (prices), OpenElectricity (generation by fuel type), Open-Meteo (weather)
