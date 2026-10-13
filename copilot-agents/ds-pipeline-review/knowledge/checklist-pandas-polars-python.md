# pandas and Python checklist

Check the project's pandas version first. Items marked **(2.x)** or **(3.x)** apply only to that version.

## Row loss and duplication
- **Merge fan-out:** `merge` on a key that isn't unique on the right (or left) side multiplies rows. Is there `validate="many_to_one"` / `"one_to_one"`, a deduplication step first, or a row-count check afterwards?
- **Unintended inner join:** `merge` defaults to `how="inner"`, which silently drops unmatched rows. Was a left join intended? `indicator=True` shows which rows matched.
- **NaN keys match each other in pandas**, unlike in SQL. Rows with a missing key on both sides join to each other, sometimes creating a large many-to-many block.
- **Inconsistent reads of the same key:** one file read with `dtype={"id": str}` and another read with default inference (so numeric-looking IDs become `int64`). The merge then raises an error, or `"007"` silently fails to match `7`. Check every reader of a key column (main input, side files such as future features, lookups, previous outputs) whenever ID formats can be numeric-looking.
- **Key mismatches:** join keys with different dtypes (`int64` vs `object`, or `"001"` vs `1`), or stray whitespace or case differences, match nothing. The merge "works" and returns NaN columns.
- **Which duplicate is kept:** `drop_duplicates()` without `subset` and `keep`, or with them after a sort that has ties, keeps an arbitrary row. Single-column `sort_values` isn't stable by default, so ties may come out in a different order. Is the row kept the intended one (for example the latest record)?
- **`concat` side effects:** frames with different columns get NaN-filled columns. Without `ignore_index=True`, duplicate index labels break later `.loc` lookups and index-aligned assignments.
- **`dropna()` without `subset`** drops a row for a NaN in any column, including columns that don't matter.

## NaN and missing values
- **`groupby` drops NaN keys** by default (`dropna=True`), so those rows vanish from the aggregates. `value_counts()` drops NaN too.
- **`groupby().first()` / `.last()`** return the first non-null value per column. The combined row may not exist in the data (column A from one row, column B from another). Use `sort_values` + `drop_duplicates`, or `nth(0)`, to get whole rows.
- **Aggregates and NaN:** the `sum()` of an all-NaN group is `0`, not NaN, while `mean()` and `count()` skip NaN. Is that the intended meaning for this metric?
- **Comparisons with NaN:** `df[col] == np.nan` is always False, so use `isna()`. `np.where(cond, a, b)` with a NaN in `cond` takes the False branch.
- **Integers and NaN:** `astype(int)` raises on NaN. A single NaN makes an integer ID column float, so `123` becomes `123.0`, which then fails to match in string joins and string comparisons. Use nullable `Int64`, or cast after filling.
- **(2.x)** `astype(str)` turns NaN into the string `"nan"`, which then passes `notna()` checks and joins as a real value.
- **Coercion hides bad data:** `pd.to_numeric(errors="coerce")` and `pd.to_datetime(errors="coerce")` silently turn unparseable values into NaN or NaT. Is the number of new NaNs checked?
- **Filling changes the meaning:** `fillna(0)` on a column where missing means "unknown" rather than "zero" biases sums and means.

## Assignment, copies and alignment
- **Chained assignment** (`df[df.a > 0]["b"] = 1`, or `df["b"][mask] = 1`) may not write back. **(3.x)** It never writes back, and there's no warning. Use `df.loc[mask, "b"] = ...`.
- **Mutating an input:** a function that adds columns to, or `inplace=`-modifies, a DataFrame it received changes the caller's data. That matters when the caller reuses the frame (for example the same raw frame feeding both train and test).
- **Index alignment:** assigning a Series or doing arithmetic between Series aligns on index labels, not position. After `reset_index`, `sort_values`, filtering or `groupby`, this silently produces NaN or puts values on the wrong rows. Watch for `df["x"] = other_df["y"]` where the two frames have different indexes.
- **`transform` vs `agg`:** `groupby().agg()` returns one row per group, while `transform()` returns one row per input row. Assigning an `agg` result back as a column produces NaN or misaligned values.

## Types, dates and times
- **Load-time type inference:** `read_csv` and similar readers drop leading zeros (zip codes, account numbers) and make an ID column float as soon as one value is missing. Are `dtype=` / `converters=` set for identifier columns?
- **Date parsing:** `pd.to_datetime` without `format=` can swap day and month (`01/02/2024`), and may parse different rows differently.
- **Integer dates:** `pd.to_datetime` on an integer column of `YYYYMMDD` values (`20240101`) treats them as nanoseconds since 1970 and returns `1970-01-01 00:00:00.020240101`. `read_csv(parse_dates=...)` parsed the same values correctly, so replacing one with the other is a silent regression. Use `format="%Y%m%d"`, or convert the values to strings first. CSV exports and SQL `INT` date keys commonly look like this.
- **Renaming into an existing name:** `rename` that maps a column to a name already present (for example mapping `ds` to `date` when the data also has a `date` column) creates duplicate column labels. `df["date"]` then returns a DataFrame, and the later code fails or picks the wrong column.
- **Naive vs aware timestamps:** mixing timezone-naive and timezone-aware timestamps, or converting to UTC in one place but not another, shifts events across day boundaries. `tz_localize` (set a timezone) and `tz_convert` (change it) are easy to confuse.
- **Boundaries:** `< end_date` vs `<= end_date` with timestamps that have a time part. `<= "2024-12-31"` excludes everything after midnight on the 31st. `between` is inclusive on both ends.
- **Week and period definitions:** `resample("W")` uses weeks ending Sunday, with buckets labeled by their end date (`label="right"`, `closed="right"`). That differs from Snowflake's `DATE_TRUNC('week', ...)`, where weeks start on Monday by default. It matters when results from the two systems are compared or joined.
- **(3.x)** `pd.to_datetime` on strings now gives microsecond resolution (`datetime64[us]`). Converting to integers with `astype("int64")` yields values 1000x smaller than before, which breaks epoch arithmetic or hashing based on nanoseconds.
- **(3.x)** `pd.offsets.Day` is a calendar day, not 24 hours, which changes results across DST transitions. The aliases `M`, `Q`, `Y` (and `BM`, `BQ`, `BY`) were removed in favor of `ME`, `QE`, `YE`.
- **(3.x)** Strings are inferred as the `str` dtype, so checks like `df[col].dtype == object` are now False, and the code that relies on them silently skips those columns.
- **Categoricals:** with `observed=False` (the default before 3.0), `groupby` on categoricals produces every combination of categories, with `0` for the `sum` of empty groups. A new category at inference time can break encoders or become NaN.
- **Money as floats:** `==` on floats, and float rounding, in money columns. Totals that should reconcile may be off by cents.

## Scale (Low severity, report only with evidence that data sizes make it matter)
- `iterrows`, `apply(axis=1)` or Python loops over rows on large frames where a vectorized operation exists
- Growing a DataFrame inside a loop (`concat` or `append` on every iteration)
- Loading a whole table into memory when the filtering or aggregation could happen in SQL

## Polars (when the code uses it, or converts between Polars and pandas)
- **NaN is not null:** `is_null()`, `fill_null()` and `drop_nulls()` ignore floating-point NaN. A column holding NaN (for example after a pandas conversion or a division by zero) passes "no nulls" checks. Use `is_nan()` / `fill_nan()`, or convert NaN to null.
- **Null join keys don't match by default** (unlike pandas, where NaN keys match each other). Rows with a null key silently fall out of inner joins. Joins also default to `how="inner"`, and `validate="m:1"` / `"1:1"` is available to catch fan-out.
- **`group_by` order isn't guaranteed** unless `maintain_order=True` is set. Code that assumes the output order (zipping it with another list, `head(1)` as "the first group", positional joins) gets a different order from run to run.
- **Window functions need sorting:** `shift`, `diff`, `cum_sum` and rolling expressions with `.over("id")` follow the current row order. Without a prior sort by (id, time), lags come from the wrong rows.
- **Non-strict casts and parses:** `cast(..., strict=False)` and `str.strptime(..., strict=False)` silently turn unparseable values into null.
- **Converting to pandas:** `to_pandas()` turns integer columns with nulls into float (with NaN), and dates and categoricals may arrive as different dtypes. See "Integers and NaN" above.
- **Lazy frames:** each `collect()` re-runs the whole query. Moving a filter before a group-by or across an outer join changes the result, not only the speed.

## The Python around the pipeline
- **Swallowed errors:** `except Exception: pass`, or a handler that returns an empty or default DataFrame, lets a failed load or API call continue downstream as if it were complete. The result is a silently empty or partial table.
- **Falsy fallbacks:** `x = param or default` replaces valid values like `0`, `0.0`, `False` or `""`. This is common with thresholds, fill values and offsets (`lag = config.get("lag") or 7` ignores `lag=0`).
- **Unhandled absence:** a lookup, env var or config key that might be missing is used without a check, so `None` flows into a filter or a query string.
- **Iterator exhaustion:** generators such as `fetch_pandas_batches()`, file handles and `iter(...)` objects are consumed once. A second loop over them silently does nothing.
- **Multiprocessing without a `__main__` guard:** a script that creates a `multiprocessing.Pool` / `ProcessPoolExecutor` (or a joblib loky backend) at module level, with no `if __name__ == "__main__":` guard, fails or re-runs the whole pipeline in every worker on Windows and on macOS (spawn start method).
- **Mutable defaults** (`def f(cols=[])`) shared across calls. Report this only when no configured linter rule already catches it (ruff/flake8-bugbear `B006`).
- **Partial writes:** several writes (files, tables, the model registry) without a transaction or an idempotent design leave inconsistent state when one step fails. See "Transactions and reruns" in checklist-sql-snowflake-dbt.md.
