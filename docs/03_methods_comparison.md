# Comparing the Five Forecasting Methods

The thesis (§4.4) compares Moving Average, SES, and Linear Regression. This doc extends that comparison to all five methods actually implemented in the code, since SARIMAX and XGBoost are runnable but currently undiscussed (see `04_thesis_review_notes.md` for the recommendation to fold this into Chapter 4).

## At a glance

| | Moving Average | SES | SARIMAX | Linear Regression (OLS) | XGBoost |
|---|---|---|---|---|---|
| Code location | `forecasting.moving_average` | `forecasting.ses_forecast_store_stats` | `forecasting.forecast_sarimax` | `regression.run_rossmann_regression` | `ml.py` |
| Inputs used | Sales history only | Sales history only | Sales history only | Sales + Promo, SchoolHoliday, CompetitionDistance, calendar | Sales + same features as OLS |
| Handles trend | Partially (lags) | Partially (lags less) | Yes (differencing) | Only via engineered month/season flags | Yes, implicitly |
| Handles weekly seasonality | Implicitly, via window=7 | No | Yes (`s=7`) | Yes, via `DayOfWeek` feature | Yes, via `DayOfWeek`/`WeekOfYear` |
| Knows about promos/holidays in advance | No | No | No | Yes | Yes |
| Interpretable | Very (it's an average) | Very (one parameter, α) | Somewhat (AR/MA coefficients) | Yes (coefficients + p-values) | Low (feature importances only) |
| Per-store fit cost | Trivial | Low | Moderate–high (1,115 independent fits) | Low | Moderate (boosted trees) |
| Train/test split | N/A (no holdout) | N/A (fit on full history) | N/A (forecasts past end of series) | Random 80/20 | **Time-ordered** 80/20 |
| GPU support | N/A | N/A | N/A | N/A | Yes, auto-detected |
| Persisted model | No | No | No | No | Yes (`rossmann_model.pkl`, via `train_xgboost_model`) |
| Dedicated backtest | No | No | No | No | Yes (`backtest_last_30_days`) |
| Failure handling | N/A | N/A | Silent `NaN` fallback per store | N/A | N/A |

## What the thesis's own numbers show (Store 1, from §4.3.2)

The thesis reports concrete SES metrics for Store 1: **MAE ≈ €659, RMSE ≈ €869, MAPE ≈ 14.36%, correlation ≈ 0.536**. It does not report equivalent Moving Average or OLS numbers for the same store in the body text — the comparison in §4.4 is currently qualitative ("SES adapts faster than MA," "regression can use information MA/SES can't") rather than a side-by-side numeric table. Running `moving_average`, `ses_forecast_store_stats`, and an OLS/RMSE print for the same store (e.g. Store 1) and putting the MAE/RMSE/MAPE side by side would make §4.4's qualitative claims falsifiable/verifiable rather than asserted.

## Conceptual grouping

**Pure time-series (history-only):** Moving Average, SES, SARIMAX. These three only ever look at past `Sales` values. Their ranking by sophistication is MA < SES < SARIMAX (each captures more structure: MA none, SES exponential recency weighting, SARIMAX explicit trend + seasonal terms), and that ordering should generally show up as decreasing error — but SARIMAX is also the most fragile of the three in this codebase (per-store convergence failures are swallowed silently), so a fair comparison needs to also report *how many stores* it failed to fit for, not just the error on the stores where it succeeded.

**Feature-driven (causal/explanatory):** OLS Linear Regression, XGBoost. Both use Promo/SchoolHoliday/CompetitionDistance/calendar features, so both can, in principle, anticipate a promotion's effect before it happens — something none of the three time-series methods can do. OLS trades some accuracy for interpretability (readable coefficients — the thesis leans on this for its "regression explains *why*" argument); XGBoost trades interpretability for accuracy (nonlinear interactions like Promo-on-a-weekend vs Promo-on-a-weekday, which a linear model can only capture if such an interaction term is hand-added, which it currently isn't).

## Suggested comparison to add to the thesis

1. Run all five methods on the **same store and the same held-out period** and report MAE/RMSE/MAPE for each in one table — currently only SES has a fully worked numeric example in the text.
2. Report SARIMAX's per-store failure rate (how many of the 1,115 stores raised and got `NaN`-filled) as a stated limitation, since it directly affects whether SARIMAX's aggregate accuracy claim is trustworthy.
3. Distinguish the two "Linear Regression" implementations (`run_rossmann_regression` vs `predict_sales_filtered`/`predict_sales_for_store`, see `02_forecasting_methods.md` §4) — the thesis's Chapter 4.3.3 narrative and pipeline diagram match `run_rossmann_regression` specifically; the other two are simpler 2-feature utilities and would only confuse the comparison if conflated with it.
4. If XGBoost and SARIMAX are added to the write-up, note that XGBoost's RMSE/MAE are computed after `expm1` back-transformation from log-space — call this out explicitly so a reader doesn't assume it's directly reading raw-Sales-space errors without transformation, which is a common source of apples-to-oranges comparisons in the forecasting literature.
