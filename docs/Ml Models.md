# ML Models

Three models were built on top of the 3-year Victorian electricity dataset — one to predict wholesale price, one to forecast demand, and one to flag spike risk. All three are deployed as a live REST API and their predictions feed directly into the AI Price Intelligence page of the dashboard.

**API:** [https://energy-models.onrender.com/docs](https://energy-models.onrender.com/docs)

---

## Why Three Models?

Electricity prices are hard to model with a single approach. Under normal conditions the market is fairly predictable — demand follows daily and seasonal rhythms, renewables follow the sun and wind. But the distribution has fat tails. Prices swing from -$461/MWh to $16,600/MWh, and spikes only happen 6.6% of the time but dominate the variance. One model trying to handle all of that ends up doing none of it well — hence three focused models, each with a specific task.

---

## Data

| Source | What it provides | Records |
|--------|-----------------|---------|
| AEMO 5-minute dispatch | Spot price, demand, generation mix | 313,344 |
| OpenElectricity API | Coal, gas, hydro, wind, solar | 196,368 |
| Open-Meteo weather archive | Temperature, humidity, wind, solar radiation | 26,376 |
| **Combined hourly dataset** | **All features joined** | **26,156** |

---

## Feature Engineering

All three models share the same feature engineering base.

**Cyclical time encoding** — Hour, day of week, and month are encoded as sine/cosine pairs rather than raw integers. This preserves their circular nature — hour 23 is actually close to hour 0, not far from it.

**Lag features** — Past values at 1h, 2h, 24h, 48h, and 168h (one week) for price, demand, temperature, renewables, and generation by fuel. The market has memory — what happened yesterday at this time is genuinely predictive.

**Rolling statistics** — 4-hour and 24-hour rolling means and standard deviations of price and demand capture short-term momentum and volatility.

**Interaction features** — A few physically meaningful combinations were added explicitly: temperature × peak flag, temperature × lagged demand, solar radiation × cloud cover.

**Net load** — Demand minus wind and solar generation represents the dispatchable generation requirement. When net load pushes against available thermal capacity, prices spike. This was one of the more impactful features for the spike model.

**Seasonal stress flags** — Four binary flags built from domain knowledge about the Victorian market:
- `summer_heatwave` — hot day with surging AC load
- `summer_stress` — hot day + low renewables (the summer paradox in feature form)
- `winter_low_wind` — cold periods when wind drops below the 25th percentile
- `high_pressure_proxy` — anticyclonic conditions suppressing both wind and solar

**Spike history** — Rolling counts of recent spikes capture their clustering behaviour. Once a spike starts, the conditions that caused it tend to persist.

**Train/test split** — Chronological 80/20 split across all three models. No random shuffling — that would leak future data into training and make performance metrics meaningless.

---

## Hyperparameter Tuning — FLAML AutoML

All three models were hyperparameter-tuned using **FLAML** (Fast and Lightweight AutoML by Microsoft Research). Rather than a grid search, FLAML uses an economical search strategy that intelligently explores hyperparameter space within a fixed time budget — evaluating XGBoost, CatBoost, and LightGBM candidates simultaneously and selecting the best algorithm and configuration for each task.

```python
from flaml import AutoML

automl = AutoML()
automl.fit(
    X_train, y_train,
    task="regression",        # or "classification" for spike model
    time_budget=600,          # 10 minutes per model
    metric="mae",
    estimator_list=["xgboost", "catboost", "lgbm"],
    eval_method="cv",
    n_splits=5
)
```

The best hyperparameters were then hardcoded for reproducibility and the final models retrained from scratch.

---

## Model 1 — Price Prediction (XGBoost)

**Task:** Predict wholesale electricity spot price ($/MWh) for the next hour

Prices are heavily right-skewed — most hours sit between $0 and $150 but rare spikes drag the distribution out to $16,600/MWh. Training a regression model on raw prices causes it to overfit to those spikes while underperforming everywhere else. The fix was a shifted log transform on the target before training:

```python
shift = abs(y_train.min()) + 1   # = 462.5, absorbs negative prices
y_train_log = np.log1p(y_train + shift)
# Inverse at inference: np.expm1(prediction) - shift
```

This compresses the extreme range and lets the model learn the normal price distribution properly, without being dominated by outliers.

**Hyperparameters:**

| Parameter | Value |
|-----------|-------|
| n_estimators | 457 |
| max_depth | 11 |
| learning_rate | 0.0259 |
| subsample | 0.841 |
| colsample_bytree | 0.627 |
| reg_alpha | 0.00561 |
| reg_lambda | 2.030 |

**Results:**

| Metric | Value |
|--------|-------|
| R² | 0.89 |
| MAE | $14.90/MWh |
| RMSE | $25.08/MWh |
| MAPE | 49.7%* |

*MAPE is inflated by hours where actual price is near zero — a $5 error on a $2/MWh hour becomes 250% MAPE. MAE and R² are the meaningful metrics here.*

**Top features:** `price_lag1`, `price_lag2`, `price_roll4_mean`, `price_range`, `temp_x_demand`, `solar_x_cloud`

---

## Model 2 — Demand Forecasting (CatBoost)

**Task:** Predict electricity demand (MW) for the next hour

Demand follows strong daily and weekly cycles driven by human behaviour, with weather effects layered on top. FLAML selected CatBoost over XGBoost for this task — it handles the smooth, structured nature of demand patterns better, and requires less feature preprocessing.

After training, feature importance analysis showed 15 features explained 98% of variance. The model was retrained on just those 15 features, which improved generalisation and reduced overfitting risk.

**Hyperparameters:**

| Parameter | Value |
|-----------|-------|
| n_estimators | 8,192 |
| learning_rate | 0.0413 |
| depth | 6 |
| l2_leaf_reg | 5.0 |
| early_stopping_rounds | 21 |

**Results:**

| Metric | Train | Test |
|--------|-------|------|
| R² | 0.99 | 0.98 |
| MAE | 68 MW | 112 MW |
| MAPE | 1.5% | 2.5% |

**Top features:** `demand_mw_lag1` (64%), `hour_cos` (10%), `hour_sin` (9%), `demand_mw_lag24` (3%)

The dominance of `demand_mw_lag1` at 64% importance tells an interesting story — electricity demand is highly autocorrelated. The best single predictor of next hour demand is simply this hour's demand, adjusted for time of day and weather. The model essentially learns to track momentum with weather and time corrections applied on top.

---

## Model 3 — Price Spike Classifier (XGBoost + Platt Calibration)

**Task:** Classify whether the next hour will be a price spike (> $200/MWh)

Spikes happen in only 6.6% of hours — a 12.4:1 class imbalance. Without intervention, the model would just predict "no spike" for everything and be right 93% of the time while being completely useless.

**Class imbalance handling:**
- `scale_pos_weight = 12.4` — upweights the spike class in XGBoost's loss function
- Sample weighting — spike hours receive 12.4× weight during training

SMOTE (synthetic oversampling) was deliberately skipped. Electricity spikes are driven by specific physical conditions — generator outages, interconnector trips, sudden demand surges. Interpolating between spike examples would generate physically meaningless data that teaches the model incorrect patterns.

**Platt calibration** was applied after training to make the raw probabilities interpretable as true likelihood estimates:

```python
from sklearn.calibration import CalibratedClassifierCV

platt = CalibratedClassifierCV(model_spike, method='sigmoid', cv='prefit')
platt.fit(X_train, y_train)
```

**Threshold tuning:** The default 0.5 decision threshold was moved to 0.7 — optimising for precision (fewer false alarms) over recall. At this threshold, 4 in 5 alerts are genuine spikes.

**Hyperparameters:**

| Parameter | Value |
|-----------|-------|
| n_estimators | 670 |
| max_depth | 10 |
| learning_rate | 0.0293 |
| subsample | 0.841 |
| scale_pos_weight | 12.4 |

**Results at threshold 0.7:**

| Metric | Value |
|--------|-------|
| ROC-AUC | 0.99 |
| Average Precision | 0.77 |
| Precision | 79% |
| Recall | 58% |
| F1 | 0.67 |

**On the 58% recall:** Most missed spikes sit in the $200–$300 range and are driven by unpredictable generator outages or interconnector trips — information that simply isn't in the feature set. The model catches 100% of extreme spikes above $500/MWh, which are the operationally critical ones.

**Top features:** `spike_lag1` (12%), `price_lag1` (11%), `demand_mw` (9%), `spike_roll24_sum` (8%), `net_load` (5%)

The fact that `spike_lag1` is the top feature at 12% confirms that spikes cluster in time — once conditions that cause a spike emerge, they tend to persist for several consecutive hours.

---

## Why the Models Don't Overfit

**Train/test gap** — All models were evaluated on the chronologically last 20% of data (roughly August 2025–February 2026), which includes seasonal conditions not seen during training. The demand model shows a train/test R² gap of 0.012 — well within the acceptable threshold of 0.05.

**Residuals over time** — Residual plots show random scatter throughout the test period with no systematic drift. An overfit model would show increasing error as the test period moves further from the training distribution — none of these models show that.

**Consistent performance across demand tiers** — The demand model maintains MAE of 100–175 MW across all demand levels from 0–8,000 MW. Only extreme heatwave demand above 8,000 MW shows higher error (440 MW), which is expected given how rarely those conditions appear in training data.

**Regularisation** — All XGBoost models include L1 and L2 regularisation terms selected by FLAML, penalising complexity and preventing the model from memorising training noise.

---

## Live API

All three models are deployed via FastAPI on Render.

**Base URL:** `https://energy-models.onrender.com`

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/` | GET | API status |
| `/health` | GET | Health check |
| `/features` | GET | Required input fields |
| `/predict` | POST | Run all three models |
| `/docs` | GET | Interactive Swagger UI |

Visit `/docs` to try it live — paste a market snapshot and get back predicted price, predicted demand, spike probability, and a spike alert label in one call.

**Sample payload:**
```json
{
  "hour_of_day": 18,
  "day_of_week": 1,
  "month": 2,
  "price": 85.5,
  "price_max": 120.0,
  "price_min": 60.0,
  "demand_mw": 6500.0,
  "renewable_pct": 25.0,
  "coal": 2000.0,
  "gas": 1500.0,
  "hydro": 300.0,
  "wind": 800.0,
  "solar": 200.0,
  "total_generation": 6800.0,
  "semischeduled_generation": 1000.0,
  "avg_temp": 28.0,
  "avg_humidity": 45.0,
  "avg_wind_speed": 8.0,
  "solar_radiation": 150.0,
  "cloud_cover": 20.0,
  "is_peak": 1.0,
  "price_lag1": 80.0,
  "price_lag2": 75.0,
  "price_lag3": 72.0,
  "price_lag24": 65.0,
  "price_lag168": 70.0,
  "demand_mw_lag1": 6400.0,
  "demand_mw_lag2": 6300.0,
  "demand_mw_lag24": 6000.0,
  "demand_mw_lag48": 5900.0,
  "demand_mw_lag168": 6100.0,
  "renewable_pct_lag1": 24.0,
  "renewable_pct_lag24": 30.0,
  "avg_temp_lag1": 27.0,
  "avg_temp_lag24": 22.0,
  "solar_radiation_lag1": 180.0,
  "solar_radiation_lag24": 200.0,
  "wind_lag1": 750.0,
  "wind_lag24": 900.0,
  "solar_lag1": 190.0,
  "solar_lag24": 210.0,
  "coal_lag1": 2100.0,
  "gas_lag1": 1400.0,
  "total_generation_lag1": 6700.0,
  "spike_lag1": 0.0,
  "spike_lag2": 0.0,
  "spike_roll4_sum": 0.0,
  "spike_roll24_sum": 0.0,
  "price_roll4_mean": 77.0,
  "price_roll4_std": 5.0,
  "price_roll24_mean": 70.0,
  "price_roll24_std": 8.0,
  "demand_roll4_mean": 6300.0,
  "demand_roll4_std": 50.0,
  "demand_roll24_mean": 6100.0,
  "demand_roll24_std": 80.0,
  "temp_roll4_mean": 27.5,
  "temp_roll24_mean": 25.0,
  "solar_roll24_mean": 175.0
}
```

This represents a typical Melbourne summer evening — 6pm, 28°C, moderate renewables, no recent spikes.

**Expected response:**
```json
{
  "predicted_price": 96.3,
  "predicted_demand": 6510,
  "spike_probability": 0.0014,
  "spike_alert": 0,
  "spike_alert_label": "NORMAL"
}
```
