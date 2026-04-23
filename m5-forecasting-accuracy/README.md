# M5 Forecasting — California Stores

End-to-end demand forecasting pipeline built on the [M5 Forecasting Accuracy](https://www.kaggle.com/competitions/m5-forecasting-accuracy) competition dataset. The project forecasts 28-day unit sales for Walmart stores in California using gradient boosting and a two-stage model for intermittent demand.

---

## Results Summary

> DEV_MODE results: CA_1 + CA_3 stores, 3-year training window, last 28 days held out for validation. `lag_364` NaNs (first year) handled natively by LightGBM. Run with `DEV_MODE=False` on Kaggle for full CA results.

| Model | Median RMSSE | vs Seasonal Naive |
|---|---|---|
| Seasonal Naive (lag-7) | 0.958 | — |
| Rolling Mean 28 | 0.713 | −26% |
| LightGBM (19 features) | 0.718 | −25% |
| Two-Stage v2 (+ intermittency features) | **0.688** | **−28%** overall; −9% on sparse series |

RMSSE (Root Mean Squared Scaled Error) is the official M5 metric — lower is better, 1.0 = matches a naïve seasonal baseline. Median is reported as mean is skewed by intermittent series with near-zero scale denominators.

---

## Project Structure

```
01_eda.ipynb               Stationarity (ADF/KPSS), ACF/PACF, STL decomposition,
                           zero-sales analysis, price sensitivity, event effects,
                           store correlation

02_baseline.ipynb          Seasonal Naive (lag-7) and Rolling Mean 28 benchmarks
                           evaluated on the last 28-day validation window

03_model.ipynb             LightGBM regressor — feature engineering, training,
                           RMSSE evaluation, residual analysis

04_results.ipynb           Model comparison charts: overall RMSSE, by category,
                           % improvement vs naive

05_two_stage_model.ipynb   Two-stage model for intermittent demand:
                           - Regular series → direct LightGBM regressor
                           - Sparse series → P(sale) × E(qty | sale)
```

---

## Feature Engineering (03_model, 05_two_stage)

| Feature | Description |
|---|---|
| `lag_7_log`, `lag_28_log`, `lag_364_log` | Log-scale lagged sales (weekly, monthly, yearly) |
| `rolling_mean_7`, `rolling_mean_28` | Short and medium-term trend |
| `rolling_std_7` | Demand volatility |
| `rolling_zero_count_7` | Intermittency signal |
| `day_of_week`, `month`, `week_of_year`, `is_weekend`, `day_of_month` | Calendar features |
| `event_indicator`, `cat_event` | Binary event flag + event category |
| `snap_CA` | SNAP benefit day (drives food sales spikes) |
| `sell_price`, `price_change_7`, `price_ratio_28` | Price level and momentum |
| `store_id`, `dept_id`, `cat_id` | Categorical identifiers |

**Intermittency features** (05_two_stage v2):

| Feature | Description |
|---|---|
| `days_since_last_sale` | Demand interval (Croston-inspired) |
| `last_nonzero_qty` | Last observed demand size |
| `zero_streak` | Consecutive zero-sale days |
| `nonzero_rate_28` | Fraction of days with sales in last 28 days |
| `mean_nonzero_qty_28` | Mean qty on sale days over last 28 days |
| `cv_28` | Coefficient of variation (std / mean) over 28 days |

---

## How to Run

### 1. Set up environment

```bash
conda create -n ds-env python=3.11
conda activate ds-env
pip install lightgbm jupyter pandas numpy statsmodels python-docx matplotlib seaborn
```

### 2. Download data from Kaggle

```bash
kaggle competitions download -c m5-forecasting-accuracy
unzip m5-forecasting-accuracy.zip -d .
```

Required files (not committed — too large):
- `sales_train_validation.csv`
- `sales_train_evaluation.csv`
- `sell_prices.csv`
- `calendar.csv` ✓ already in repo

### 3. Run notebooks in order

```
01_eda.ipynb → 02_baseline.ipynb → 03_model.ipynb → 04_results.ipynb → 05_two_stage_model.ipynb
```

### DEV_MODE flag

All modelling notebooks have a `DEV_MODE = True` toggle at the top. When enabled, the pipeline loads CA_1 + CA_3 and 2 years of data (~2M rows) for fast local iteration. Set `DEV_MODE = False` on Kaggle for the full CA dataset (~23M rows).

---

## Key Findings

- **Lag features dominate**: `lag_7_log` and `lag_364_log` are the top predictors — weekly seasonality and year-over-year patterns drive most forecast signal.
- **SNAP days matter**: SNAP benefit days produce statistically significant sales spikes in FOODS categories (+15–20% vs non-SNAP days).
- **Intermittent demand is a distinct problem**: ~40% of CA series have >50% zero-sale days. A classifier-gated model reduces RMSSE on sparse series by separating "will there be a sale?" from "how many?".
- **Price features add marginal value**: `price_change_7` and `price_ratio_28` help in HOUSEHOLD/HOBBIES but not FOODS (prices are stickier).

---

## Repo Contents

| File | Description |
|---|---|
| `baseline_benchmark.csv` | Naive and rolling mean RMSSE scores |
| `lgb_model.txt` | Trained LightGBM model (single-stage) |
| `lgb_two_stage_*.txt` | Two-stage model files (regular, classifier, quantity) |
| `lgb_{clf,qty,reg}_v2.txt` | Two-stage v2 with intermittency features |
| `baseline_model_explanation.docx` | Plain-language explanation of modelling approach |
| `DATASETS.md` | Dataset schemas and column descriptions |
| `requirements.txt` | Python package versions |
| `ca_*.png` | All generated charts |
