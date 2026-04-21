# M5 Forecasting — Dataset Reference

## Competition Overview

The M5 Forecasting Accuracy competition involves forecasting 28 days of unit sales for 3,049 products across 10 stores in 3 US states (CA, TX, WI) using Walmart retail data.

---

## Raw Files

### `calendar.csv`
| Property | Value |
|---|---|
| Rows | 1,969 |
| Columns | 14 |
| Coverage | 2011-01-29 → 2016-06-19 (d_1 … d_1969) |

| Column | Description |
|---|---|
| `date` | Calendar date (YYYY-MM-DD) |
| `wm_yr_wk` | Walmart week number |
| `weekday` | Day name (Saturday … Friday) |
| `wday` | Day of week (1=Saturday) |
| `month` | Month number |
| `year` | Year |
| `d` | Day ID used to join with sales data |
| `event_name_1/2` | Name of sporting/cultural/national/religious event |
| `event_type_1/2` | Category of event |
| `snap_CA/TX/WI` | SNAP benefit active that day for the state (0/1) |

---

### `sell_prices.csv`
| Property | Value |
|---|---|
| Rows | 6,841,121 |
| Columns | 4 |

| Column | Description |
|---|---|
| `store_id` | Store identifier (e.g. CA_1) |
| `item_id` | Item identifier (e.g. HOBBIES_1_001) |
| `wm_yr_wk` | Walmart week — join key with calendar |
| `sell_price` | Sell price in USD for that week |

---

### `sales_train_validation.csv`
| Property | Value |
|---|---|
| Rows | 30,490 (one per item-store series) |
| Columns | 1,919 |
| Day range | d_1 … d_1913 |

First 6 columns are metadata; the rest are daily unit sales:

| Column | Description |
|---|---|
| `id` | Series ID (item + store + "_validation") |
| `item_id` | Item identifier |
| `dept_id` | Department (e.g. HOBBIES_1) |
| `cat_id` | Category (HOBBIES / HOUSEHOLD / FOODS) |
| `store_id` | Store (CA_1 … WI_3) |
| `state_id` | State (CA / TX / WI) |
| `d_1 … d_1913` | Daily unit sales |

---

### `sales_train_evaluation.csv`
Same schema as `sales_train_validation.csv` but extends to **d_1941** (+28 days), used for final evaluation scoring.

| Property | Value |
|---|---|
| Rows | 30,490 |
| Columns | 1,947 |
| Day range | d_1 … d_1941 |

---

### `sample_submission.csv`
Submission template for the competition.

| Property | Value |
|---|---|
| Rows | 60,980 (30,490 validation + 30,490 evaluation rows) |
| Columns | 29 |

| Column | Description |
|---|---|
| `id` | Series ID |
| `F1 … F28` | 28-day forecast values |

---

## Derived / Cached Parquet Files

### `train.parquet`
Fully processed, melted, and feature-engineered training dataset.

| Property | Value |
|---|---|
| Rows | 57,473,650 |
| Columns | 26 |

| Column | Description |
|---|---|
| `id, item_id, dept_id, cat_id, store_id, state_id` | Series metadata |
| `d` | Day ID (d_1 … d_1913) |
| `d_num` | Numeric day index |
| `date` | Calendar date |
| `sales` | Unit sales (target variable) |
| `wm_yr_wk, weekday, wday, month, year` | Calendar features (joined from calendar.csv) |
| `event_name_1/2, event_type_1/2` | Event features |
| `snap_CA, snap_TX, snap_WI` | SNAP flags |
| `sell_price` | Joined from sell_prices.csv |
| `sales_lag_7, sales_lag_14, sales_lag_28` | Lagged sales features |

---

### `val.parquet`
Validation split — last 28 days per series. Same schema as `train.parquet`.

| Property | Value |
|---|---|
| Rows | 853,720 |
| Columns | 26 |

---

### `df_CA_cache.parquet`
California-only subset of melted data. Used by `ca_analysis.ipynb` for state-level EDA.

| Property | Value |
|---|---|
| Rows | 23,330,948 |
| Columns | 23 |

Same columns as `train.parquet` excluding the lag features (`sales_lag_7/14/28`).

---

## Notebooks

| Notebook | Purpose |
|---|---|
| `SalesForecasting.ipynb` | Full pipeline: data melting, feature engineering, model training |
| `ca_analysis.ipynb` | California-focused EDA using `df_CA_cache.parquet` |

---

## Key Relationships

```
calendar.csv  ──(d / wm_yr_wk)──►  sales_train_*.csv
sell_prices.csv ──(store_id + item_id + wm_yr_wk)──►  sales_train_*.csv

sales_train_validation.csv  ──melt + join──►  train.parquet / val.parquet
train.parquet  ──filter state_id=CA──►  df_CA_cache.parquet
```
