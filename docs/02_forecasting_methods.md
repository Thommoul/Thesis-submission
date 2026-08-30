# Forecasting Methods — Theory and Implementation

Five forecasting approaches exist in the codebase. The thesis text (Chapter 4) currently only writes up the first three; SARIMAX and XGBoost are fully implemented and produce output files but aren't discussed in the document (see `04_thesis_review_notes.md`).

## Error metrics (used throughout)

For actual value `y_t` and forecast `ŷ_t`:

| Metric | Formula | Notes |
|---|---|---|
| Error | `y_t − ŷ_t` | signed; positive = model underestimated |
| Absolute Error (AE) | `\|y_t − ŷ_t\|` | |
| Squared Error (SE) | `(y_t − ŷ_t)²` | penalizes large misses |
| Absolute Percentage Error (APE) | `\|AE / y_t\| × 100` | scale-free, per observation |
| MAE | `mean(AE)` | same units as sales |
| MSE | `mean(SE)` | |
| RMSE | `sqrt(MSE)` | same units as sales, sensitive to outliers |
| MAPE | `mean(APE)` | % — comparable across stores/scales |
| Correlation | `corr(y, ŷ)` | direction/co-movement, not magnitude accuracy |

## 1. Moving Average — `forecasting.moving_average`

**Idea:** forecast for day *t* = mean of the previous *n* days' actual sales. A naive baseline that smooths short-term noise.

**Implementation:** 7-day rolling window (`n = 7`, chosen to span one full weekly cycle, since Rossmann sales are strongly day-of-week dependent), computed per store after sorting by `Store, Date`. Values are additionally scaled `/1000` in the `Moving_Average,n=7` output column. Error/AbsError/SqError/APE are computed against the *raw* (unscaled) rolling mean, not the scaled one.

**Strengths:** trivial to compute/explain, no parameters to fit, good at surfacing the underlying trend.

**Weaknesses:** equal weight to all *n* observations → lags behind sudden changes (promotions, holidays); can't use any information outside the sales series itself; first `n−1` rows of every store are `NaN`.

## 2. Simple Exponential Smoothing (SES) — `forecasting.ses_forecast_store_stats`

**Idea:** `ŷ_{t+1} = α·y_t + (1−α)·ŷ_t` — each new forecast is a weighted blend of the latest actual and the previous forecast, so older observations decay in influence geometrically instead of dropping out abruptly like the moving average.

**Implementation:** `statsmodels.tsa.holtwinters.SimpleExpSmoothing`, fixed `α = 0.30`, `optimized=False` (α is not fit — it's the value chosen a priori as a trade-off between responsiveness and stability, per the thesis text). Rows are filtered to `Open == 1` and `Sales > 0` first — closed days would otherwise register as demand collapses rather than the operational non-events they are (99.97% of zero-sales rows in the dataset are closed stores, per the thesis's own zero-sales analysis). Can run for one store or loop over every store.

**Strengths:** adapts faster to recent shifts than moving average; single parameter; low compute cost.

**Weaknesses:** still univariate (no Promo/holiday awareness); assumes no trend or seasonality — Holt / Holt-Winters exist for that and aren't implemented here.

**Measured example (Store 1, α = 0.30, per the thesis):** MAE ≈ €659/day, RMSE ≈ €869, MAPE ≈ 14.36%, correlation ≈ 0.536.

## 3. SARIMAX — `forecasting._fit_sarimax_forecast` / `forecast_sarimax`

**Idea:** seasonal ARIMA — models autocorrelation, differencing, and a weekly seasonal cycle explicitly, rather than only smoothing.

**Implementation:** `SARIMAX(order=(1,1,1), seasonal_order=(1,1,1,7))` fit independently per store on one calendar year of data, forecasting `forecast_days` (default 30) past the last observed date. Stores with fewer than 30 observations are skipped; if the fit raises (common with SARIMAX on short/irregular series), the store's forecast rows are filled with `NaN` rather than the run failing.

**Strengths:** can capture trend + weekly seasonality without hand-picking a window/α; standard, well-understood statistical model.

**Weaknesses:** one fit per store (1,115 stores) is comparatively expensive; still univariate — no Promo/SchoolHoliday signal; per-store convergence is not guaranteed (hence the silent `except Exception` fallback, which also means failures aren't surfaced anywhere the user would see them).

**Not currently discussed in the thesis text**, despite being implemented and runnable via `run.py --steps sarimax`.

## 4. Linear Regression

Two distinct implementations exist under this name, at different levels of sophistication — worth distinguishing clearly in any write-up:

### 4a. OLS with the full feature set — `regression.run_rossmann_regression`
`statsmodels.OLS` on `[Month, DayOfWeek, Promo, SchoolHoliday, CompetitionDistance, IsSummer, IsChristmas, IsWeekend]` (+ intercept), 80/20 split, prints the full model summary (coefficients, p-values, R²) and RMSE. This is the version that matches the thesis's Chapter 4.3.3 description and its "Raw Data → Merge → Preprocessing → Feature Engineering → 80/20 Split → OLS → Evaluation → Excel" pipeline diagram.

### 4b. Two-feature sklearn regressions — `regression.predict_sales_filtered`, `regression.predict_sales_for_store`
`sklearn.LinearRegression` on just `[DayOfWeek, Promo]` — a much smaller model, per month or per store, used to generate a lookup table of predicted sales for every `(day, promo)` combination rather than to explain variance. These are **not** the model the thesis narrates in 4.3.3; they're a separate, simpler utility that happens to share the "linear regression" name.

**Strengths (regression generally):** unlike MA/SES/SARIMAX, it can use information known *in advance* (a promo is scheduled, a date is a school holiday) rather than only trailing sales — this is the thesis's central argument for why regression outperforms the pure time-series methods operationally, not just statistically.

**Weaknesses:** OLS assumes a linear, additive relationship and independent errors — daily sales residuals are typically autocorrelated, which the current implementation doesn't test for (no Durbin-Watson / residual-autocorrelation check reported); `CompetitionDistance` and `Promo2` NaNs are median/zero-filled without sensitivity analysis.

## 5. XGBoost — `ml.py`

**Idea:** gradient-boosted trees on the same calendar/business features as the regression, but with no linearity assumption and automatic interaction/nonlinearity discovery.

**Implementation:** `XGBRegressor(tree_method="hist", n_estimators=500, learning_rate=0.05, max_depth=5, subsample=0.8, colsample_bytree=0.8)`, trained on `log1p(Sales)` (predictions are `expm1`-transformed back), using the shared `FEATURES` list from `features.py`. Split is **time-ordered**, not random — the last 20% chronologically is held out, which is the methodologically correct choice for a time series (unlike the `train_test_split(..., random_state=42)` used in the regression models, which shuffles randomly and can leak future information into training).

Three variants:
- `run_store_full_analysis(store_id)` — train + evaluate, not persisted.
- `train_xgboost_model(store_id)` — same, but persists to `rossmann_model.pkl`; feeds `backtest_last_30_days`.
- `xgboost_all_stores_model_detailed()` — one global model across all stores (`Store` id as a feature), `n_estimators=600, max_depth=6`; also produces best/worst-5-stores-by-MAE breakdowns.

**GPU:** `device.get_xgb_device()` auto-detects CUDA and falls back to CPU transparently; no manual configuration needed (`readme.md` is accurate on this point).

**Backtesting** — `backtest_last_30_days(store_id)`: scores the persisted model on the most recent 30 rows of real history, reports MAE/RMSE/MAPE, and writes an Excel dashboard with an embedded actual-vs-predicted line chart. This is the only method in the codebase with an explicit holdout-style sanity check separate from its own training split.

**Strengths:** handles nonlinearity and feature interactions (e.g. Promo × DayOfWeek) without manual specification; typically the most accurate of the five on tabular sales data; scales to a single global multi-store model.

**Weaknesses:** least interpretable of the five (no coefficients to read off, unlike OLS); more hyperparameters, more compute; log-target transform means care is needed when comparing its RMSE directly against the other methods' RMSE (should be computed post-`expm1`, which the code does correctly, but this is easy to get wrong).

**Not currently discussed in the thesis text at all**, despite being the most developed part of the codebase (own module, GPU support, model persistence, and the only method with dedicated backtesting).
