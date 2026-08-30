# Code Architecture

Overview of what the `rossmann/` package and `run.py` actually do, module by module, as of the current codebase (not the thesis narrative — see `04_thesis_review_notes.md` for where the two diverge).

## Data flow

```
data/train.csv, data/test.csv, data/store.csv
        │
        ▼
rossmann/data.py          load_data() / preprocess_data() / load_store_history()
        │
        ├──► rossmann/forecasting.py   Moving Average, SES, SARIMAX
        ├──► rossmann/regression.py    OLS regression, per-store/month Linear Regression
        ├──► rossmann/ml.py            XGBoost training, backtesting
        ├──► rossmann/analytics.py     closed-day / zero-sales / average-sales reports
        └──► rossmann/plots.py         sales density, sales growth charts
                │
                ▼
        outputs/*.csv, outputs/*.xlsx, matplotlib figures
```

`rossmann/features.py` and `rossmann/device.py` are shared helpers used by `ml.py` (and, for features, by `regression.py`'s conceptual equivalent — see note below).

## Module reference

### `config.py`
Defines `BASE_DIR`, `DATA_DIR`, `OUTPUT_DIR`, `MODEL_PATH` and creates `outputs/` if missing. Every other module imports its paths from here — there is no other place paths are hardcoded.

### `data.py`
- `load_data()` — reads `store.csv`, `test.csv`, `train.csv` as-is.
- `preprocess_data(store, test, train)` — drops `StoreType`/`Assortment` from `store`, drops `StateHoliday` from `train`/`test`, reformats `Date` to `dd/mm/yyyy` strings, and derives `Day of the week`, `Month`, `Week/52` columns. **Note:** after this function runs, `Date` is a *string*, not a `datetime` — several downstream functions (`forecasting.moving_average`, `analytics.sales_by_day_for_year`) re-parse it with `pd.to_datetime` themselves rather than relying on this preprocessing.
- `load_store_history(store_id)` — independent path: reads `train.csv` fresh, merges with `store.csv`, filters to one store and `Open == 1`. Used only by `ml.py`. Does not go through `preprocess_data`.

### `features.py`
Single source of truth for the feature set used by the ML models:
```python
FEATURES = ["Promo", "SchoolHoliday", "CompetitionDistance", "Month",
            "DayOfWeek", "WeekOfYear", "IsWeekend", "IsSummer", "IsChristmas"]
```
`add_calendar_features(df)` derives `Month`, `DayOfWeek`, `WeekOfYear`, the three boolean flags, and median-fills `CompetitionDistance`. Used by `ml.py`. `regression.run_rossmann_regression` re-implements an equivalent (but not identical) feature block inline instead of importing this — see review notes.

### `forecasting.py`
- `moving_average(train, window=7, store_id=None, save=True)` — rolling mean of `Sales`, plus `Error`, `AbsError`, `SqError`, `APE`. Writes a `;`-separated CSV.
- `ses_forecast_store_stats(alpha=0.3, store_id=None)` — `statsmodels.SimpleExpSmoothing` per store (or all stores), filtered to `Open == 1` and `Sales > 0`. Writes an Excel workbook with `Daily` and `Stats` (MAE/MSE/RMSE/MAPE/Correlation) sheets.
- `_fit_sarimax_forecast` / `forecast_sarimax(year, forecast_days)` — per-store `SARIMAX(order=(1,1,1), seasonal_order=(1,1,1,7))`, forecasts `forecast_days` ahead past the end of the filtered year, silently records `NaN` forecasts for stores where the fit raises.
- `forecast_store_sales_per_year` / `forecast_store_sales_per_month` — variants that also export the actual history alongside the forecast, or just export a raw actuals slice with no model at all (the "per_month" one doesn't forecast anything — it's a filtered export).

### `regression.py`
- `run_rossmann_regression(...)` — merges `train`+`store`, engineers `Year/Month/DayOfWeek/WeekOfYear/IsWeekend/IsSummer/IsChristmas`, fits `statsmodels.OLS` on `[Month, DayOfWeek, Promo, SchoolHoliday, CompetitionDistance, IsSummer, IsChristmas, IsWeekend]` (80/20 split), prints the full OLS summary + RMSE, optionally plots three seasonality charts.
- `predict_sales_filtered(year, month=None)` — much simpler per-month model: `sklearn.LinearRegression` on just `[DayOfWeek, Promo]`, writes one Excel sheet per month with predictions for every `(day, promo)` combination.
- `predict_sales_for_store(store_id, year=None, month=None)` — same simple two-feature model, scoped to one store, writes actual-vs-predicted Excel with MAE/RMSE printed.

### `ml.py`
- `_make_xgb_regressor(**params)` — builds an `XGBRegressor(tree_method="hist", device=...)`, device chosen by `device.py`.
- `_train_test_split_by_time(df, features, ...)` — time-ordered (not random) 80/20 split, target is `log1p(Sales)` by default.
- `run_store_full_analysis(store_id)` — trains, evaluates, returns model/results/RMSE. Does not persist the model.
- `train_xgboost_model(store_id)` — same training, but **also `joblib.dump`s `(model, None, FEATURES)` to `MODEL_PATH`** (`rossmann_model.pkl`), which is what `backtest_last_30_days` later loads. This is the only function that writes the model file.
- `xgboost_all_stores_model_detailed()` — single global model trained on all stores with `Store` added as a numeric feature, writes a 4-sheet workbook (`Global_Predictions`, `Store_Summary`, `Best_Stores`, `Worst_Stores`).
- `backtest_last_30_days(store_id)` — loads the persisted model, scores it on the last 30 rows of that store's history, writes an Excel dashboard with an embedded `openpyxl` line chart. **Depends on `train_xgboost_model` having been run first** for that store — there's no check, so running backtest cold raises a `FileNotFoundError`/unpickling error.

### `device.py`
`get_xgb_device()` — tries to fit a 2-row `XGBRegressor` with `device="cuda"`; returns `"cuda"` on success, `"cpu"` on any exception. Result is `lru_cache`d for the process.

### `analytics.py`
Standalone reporting functions, each reads its own CSV rather than taking a pre-loaded frame (only `sales_by_day_for_year` and `zero_sales_report` take a `train` argument): `sales_by_day_for_year`, `totalsalesofyear`, `check_closed_days`, `average_sales_when_open`, `average_sales_for_store`, `zero_sales_report`.

### `plots.py`
`plot_sales_density()` (KDE of sales, Promo2 vs not) and `plot_sales_growth()` (day-over-day % change + LOWESS trend line). Both call `plt.show()` — they're meant for interactive/notebook use, not headless runs.

### `run.py` — CLI entry point
Registers steps via a `@step("name")` decorator into a `STEPS` dict, currently only:
```
moving_average, ses, sales_by_day, sarimax
```
`--list` prints registered steps, `--steps a b c` runs a chosen subset, `--all` runs every registered step, with no arguments it runs `moving_average` for store 1. **Only these four of the ~15 public functions across the package are wired into the CLI** — everything in `regression.py`, `ml.py`, `plots.py`, and most of `analytics.py` is currently reachable only by importing the module directly in a script/REPL. See `04_thesis_review_notes.md` for the mismatch this creates against `readme.md`.
