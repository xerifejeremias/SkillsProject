# Metric logic and ML / forecasting checklist

These bugs live in the logic rather than the syntax. They apply equally to pandas and SQL.

## Metric and aggregation logic
- **Average of averages:** averaging precomputed rates or means across groups of different sizes (for example `groupby("region")["conversion_rate"].mean()`, or `AVG(daily_avg)`). Aggregate from the raw numerator and denominator instead: `sum(conversions) / sum(visits)`.
- **Wrong denominator:** rates computed over the wrong population, a denominator that can be zero, or a denominator filtered differently from its numerator (for example a numerator with test accounts removed over a denominator that still includes them).
- **Partial periods:** the current month-to-date or week-to-date is compared with full past periods, or the last, incomplete day appears in a trend as a drop. Is the last period excluded or labeled?
- **Gaps in time series:** days with no events produce no row after a `groupby` or `GROUP BY`, so rolling windows, `shift`, `LAG` and averages skip them silently. Is the series reindexed onto a complete calendar (date spine), with zeros where zero is correct?
- **Survivorship through joins:** inner joining facts to a table that only holds current entities (active customers, live products) drops history for churned or deleted ones.
- **Parts that don't sum to the whole:** segments that overlap, or a segment column with NULLs that falls out of the breakdown, so the segment totals don't match the overall total.
- **Mismatched definitions:** the code computes a metric differently from its name, docstring or the project's documented definition. For example, "active users" counts events rather than distinct users.

## Leakage in features and training
- **Fitting on all the data before the split:** scaling, imputation, target or mean encoding, outlier thresholds, feature selection or PCA fitted before the train/test split, or before each CV fold. Fit only on training data, ideally inside an sklearn `Pipeline` so that cross-validation refits them per fold.
- **Random splits on time-ordered data:** the test set must come after the training set in time (`TimeSeriesSplit`, or a cutoff date). A random split of time-ordered data inflates the metrics.
- **The same entity on both sides:** rows from one customer, patient or store in both train and test, when the model will predict for new entities. Use `GroupKFold` / `GroupShuffleSplit`.
- **Future information in features:**
  - `shift(-1)`, `LEAD`, or centered rolling windows (`center=True`)
  - Rolling or expanding windows that aren't sorted by time, or not grouped per entity, so values bleed across entities
  - Aggregates over the full history used as features for earlier rows
  - Columns that are filled in after the outcome happens (for example `cancellation_reason` when predicting churn)
- **Out-of-fold rules for encoders and stacking:** target or mean encoding, and any "learned" aggregate feature, must be computed out of fold for the training rows. A stacking or blending meta-model must be trained on the base models' out-of-fold predictions, never on their in-sample predictions. Otherwise the meta-model learns to trust overfitted base models.
- **Overlapping windows across the split:** with lag or rolling features and h-step-ahead targets, consecutive rows share information. A time-based split or CV fold without a gap of at least the horizon (and the longest feature window, when windows look forward) leaks across the boundary. Check `TimeSeriesSplit(gap=...)`, or purge and embargo.
- **Nested CV with a different split rule:** when hyperparameters are tuned inside an outer evaluation, the inner splits must follow the same grouping or time rule as the outer ones. A random inner `KFold` inside a time-based or grouped outer split leaks.
- **Held-out actuals passed as history:** calling `predict` / `forecast` with the test period's actual target values as the history or `new_df` (or updating the model state with them) before forecasting that same period.
- **Tuning on the test set:** choosing hyperparameters, thresholds or the "best" model on the test set and then reporting that test score as the final estimate.
- **Train/serve mismatch:** feature code that differs between training and inference, such as different fill values, category mappings, column order or timezone handling. Also check that the encoders and scalers fitted at training time are saved and reused at inference rather than refitted.

## Forecasting and backtesting
- **Features unavailable at forecast time:** exogenous features (weather, prices, promotions, other series) that use actual values the model wouldn't have at the forecast origin. They should be lagged by at least the forecast horizon, or replaced by the forecast of the feature.
- **Lags shorter than the horizon:** a lag of 1 used for a 7-day-ahead forecast in direct multi-step forecasting. In recursive forecasting, check that predictions, not actuals, are fed back as inputs.
- **Backtest leakage:** the backtest refits or tunes on data after each origin, or the evaluation window overlaps the training window. Check that each fold's training data ends strictly before that fold's forecast origin.
- **Pooling errors across series of different sizes:** model selection, ensembling weights or "global best model" rules that add up or average scale-dependent errors (MAE, RMSE, MSE) across series. The largest series then decide for everyone, and a model that is much worse on most small series can win. Use scale-free comparisons, such as the mean of each series' error ratio to a baseline, MASE, or rank-based voting. Check what the default metric is. This matters most when the pooled choice is then applied back to the individual series.
- **Selection and evaluation on the same folds:** the score of the chosen model or ensemble reported on the same backtest folds used to choose it is optimistic. Report this only when that score is presented as an out-of-sample estimate.
- **Misleading error metrics:** MAPE on series with zeros or near-zero values gives division by zero or explodes. Metrics averaged across series of very different scales are dominated by the largest series.
- **Target transform vs metric:** training on `log1p(y)` and then scoring or reporting on `expm1(pred)` is fine for RMSLE, but `expm1` of a log-scale forecast estimates the median, not the mean. That biases sums and totals low. Negative predictions passed to RMSLE or a log metric give NaN or errors unless they are clipped. Also check that the inverse transform matches the forward one (the same offset, the same differencing order, the same per-series scale).
- **Metrics on misaligned arrays:** `y_true` and `y_pred` coming from frames sorted or filtered differently, then passed to a metric as arrays, compare different rows. Align on (id, date) before computing errors. The same applies to writing submissions or outputs back to IDs by position.
- **Frequency and calendar:** resampling with a different week or month definition than the business uses (see "Week and period definitions" in checklist-pandas-polars-python.md), DST days with 23 or 25 hours in hourly data, and holiday calendars for the wrong country.

## Classical statistical models (statsmodels, ARIMA, ETS, pmdarima)
- **Double differencing:** passing an already-differenced series to `ARIMA(order=(p, d, q))` with `d > 0` (or a seasonal `D > 0` on seasonally differenced data) differences twice. Forecasts then have to be integrated back the right number of times.
- **One-step predictions used as multi-step accuracy:** in-sample or static predictions (`predict(start, end)` within the observed range, `dynamic=False`, `fittedvalues`) use the actual values at every step. Reporting them as a multi-step forecast error overstates accuracy. Multi-step evaluation needs `dynamic=True`, `forecast()` from each origin, or rolling refits.
- **Exogenous variables:** future exog rows must cover every forecast step, in the same column order and with the same transforms as at fit time. Realized future values of exog (actual prices, weather) used in a backtest leak unless they would be known at forecast time.
- **Model selection loops:** a grid search that keeps the last fitted model instead of the best, compares information criteria (AIC/BIC) across models fitted on differently differenced or transformed data (not comparable), or silently skips non-converged fits without recording them.
- **Seasonal period:** a `period` / `seasonal_periods` / `m` that doesn't match the data frequency (12 for monthly, 7 or 365.25 for daily, 52 for weekly), or that's hard-coded while the frequency is configurable.

## Probabilistic forecasts and prediction intervals
- **Adding up quantiles:** summing per-series or per-step quantiles (q90 of each item added up to a "q90 of the total") isn't a quantile of the sum. The same holds for medians. Point means can be added up, but quantiles can't. Also check hierarchical reconciliation applied to quantiles.
- **Interval labels and indexes:** a q10–q90 band labeled as a 90% interval (it's 80%), quantile arrays indexed off by one (some libraries put the mean at index 0), and quantiles that cross (q10 > q50) without sorting or fixing.
- **Fallbacks for new or unseen series:** a series that wasn't seen during fit (a new item at prediction time, or one with too little history) often gets a fallback: a global average residual std, a global mean forecast, a "dummy" forecast, or a constant scale of 1.0. Check that the fallback is on the right scale for that series. A pooled average across series, or a value computed in scaled units and then applied in original units (or the reverse), gives that series intervals or forecasts of the wrong size. Prefer a per-series estimate from its own history (for example its seasonal-naive error). Check this especially when the change touches scaling or normalization.
- **Intervals from in-sample residuals:** conformal or empirical intervals calibrated on training-fit residuals rather than on out-of-sample (backtest) errors are too narrow. Also check that intervals are computed per horizon step when the error grows with the horizon.
- **Clipping that breaks the intervals:** clipping point forecasts at zero but not the quantiles (or the reverse), or clipping series that can legitimately be negative (returns, temperatures, net flows).

## Neural network model code (PyTorch and similar)
- **Train/eval mode:** functional calls such as `F.dropout(x, p)` without `training=self.training`, or batch norm used in functional form, ignore `model.train()` / `model.eval()`. Depending on the library version, dropout then never applies, or it also applies at prediction time and makes predictions random. Also check that `model.eval()` and `torch.no_grad()` are used for validation and prediction.
- **Reshapes that mix the batch dimension:** `view` / `reshape` across dimensions laid out in a different order (for example `(phase*B + b)` read back as `(b*phase + k)`) silently mixes values from different samples. Predictions then depend on what else is in the batch. Check the memory order before each `view`, and use `permute(...).contiguous()` where the order changes. A small index-tracking check with integer tensors settles it.
- **Scale mismatches in fed-back inputs:** with teacher forcing or recursive decoding, the values fed back at inference must be on the same scale and transform as the teacher-forced inputs seen in training (standardized, clipped, log-transformed).
- **Initialization or configuration that never takes effect:** loops over `parameters()` checked with `isinstance(m, nn.Linear)` (parameters are tensors, not modules), misspelled parameter names behind `hasattr` checks, or init applied to tensors that a hook recomputes on every forward pass.
- **Artifacts saved at the wrong moment:** weights, scalers or encoders saved before training or fitting (for example `save_weights` right after building the model, then loaded later as a "trained" checkpoint), or saved before the final refit, so prediction uses an older or untrained state.
- **Checkpoint averaging and selection:** picking or averaging checkpoints by validation loss, then reporting that same validation score as the estimate. Also checkpoint file names that can collide and overwrite each other.

## Reproducibility (report only when it changes results that are compared or published)
- Random seeds not set for splits, sampling or models whose results are compared across runs
- Results that depend on row order that SQL or a parallel load doesn't guarantee: a `LIMIT` or `QUALIFY` without a unique `ORDER BY`, or a `groupby().first()` after an unordered load
