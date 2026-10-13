# SQL, Snowflake and dbt checklist

## Joins and grain
- **Join fan-out:** joining to a table with several rows per key and then using `SUM` or `COUNT` double-counts. Common signs are a `DISTINCT` added to hide duplicates, `COUNT(*)` where `COUNT(DISTINCT id)` was meant, or aggregating a parent-level measure (an order total) after joining to child rows (order lines).
- **Outer join undone by WHERE:** `LEFT JOIN b ... WHERE b.col = 'x'` turns the left join into an inner join. Put the condition in the `ON` clause, or allow `b.col IS NULL`.
- **Join to the current dimension:** joining facts to a dimension's current state (customer segment, region, price) rather than its state at the time of the event rewrites history. Is a validity-dated (SCD2) join needed?
- **GROUP BY grain:** check that `GROUP BY` matches the intended grain. Watch for `ANY_VALUE` or `MAX` used to dodge grouping on a column that actually varies within the group.
- **`UNION` vs `UNION ALL`:** `UNION` removes duplicate rows, which drops legitimate identical rows. `UNION ALL` keeps them. Is the one used intended?

## NULL semantics
- `NOT IN (subquery)` returns no rows at all if the subquery contains a NULL. Use `NOT EXISTS`.
- `col != 'x'` and `col NOT LIKE ...` exclude rows where `col` is NULL.
- `COUNT(col)` skips NULLs, while `COUNT(*)` doesn't. `AVG` ignores NULLs, so it averages over the non-NULL rows only.
- `NULL = NULL` isn't true, so joins on nullable keys drop those rows. Snowflake has `EQUAL_NULL()` and `IS [NOT] DISTINCT FROM` for NULL-safe comparison.
- `CONCAT` and `||` return NULL if any argument is NULL. `CONCAT_WS` and `COALESCE` are the usual fixes.
- Division by zero: use `x / NULLIF(y, 0)` or `DIV0` / `DIV0NULL`, and decide whether the result should be NULL or 0.

## Windows, dates and types
- **Default window frame:** with `ORDER BY` and no explicit frame, aggregate window functions use `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`. Ties in the `ORDER BY` share the same running total. Use an explicit `ROWS` frame for a row-by-row running total.
- **`FIRST_VALUE` / `LAST_VALUE` / `NTH_VALUE`** default to the whole partition in Snowflake. That isn't ANSI behavior, so code ported from other databases can change meaning.
- **Nondeterministic picks:** `ROW_NUMBER()` / `QUALIFY ... = 1` with an `ORDER BY` that isn't unique picks an arbitrary row among the ties, which can differ between runs. Add a tie-breaker column.
- **`LAG` / `LEAD`** without the right `PARTITION BY` read values across entities, for example from the previous customer.
- **Date boundaries:** prefer half-open ranges (`>= start AND < end`). `BETWEEN '2024-01-01' AND '2024-12-31'` on a timestamp drops everything after midnight on Dec 31. Also check which timezone the boundary is in.
- **Implicit casts:** comparing strings to dates or numbers (`'2024-01-01' < ts_col`, `id = '0012'`) relies on implicit conversion and may drop leading zeros. Integer and decimal arithmetic can lose scale.

## Snowflake specifics
- **Timestamp types:** `TIMESTAMP_NTZ`, `TIMESTAMP_LTZ` and `TIMESTAMP_TZ` behave differently. `LTZ` depends on the session `TIMEZONE` parameter, and `CURRENT_TIMESTAMP()` returns `LTZ`. `TIMESTAMP` is an alias set by `TIMESTAMP_TYPE_MAPPING`, which defaults to `NTZ`. Mixing these types, or relying on session defaults that differ between users, the scheduler and the connector, shifts values by hours.
- **Identifiers and case:** unquoted identifiers are uppercased. `"myCol"` is case-sensitive and is a different column from `mycol`. The connector and Snowpark return column names in uppercase, so pandas code that expects lowercase names raises `KeyError`, or worse, creates a new column instead of updating the existing one.
- **`MERGE` with duplicate source keys:**
  - On the update/delete branch, duplicate source keys cause an error by default (`ERROR_ON_NONDETERMINISTIC_MERGE = TRUE`). If someone set that parameter to FALSE, an undefined row wins.
  - On the `WHEN NOT MATCHED THEN INSERT` branch, every duplicate source row is inserted, silently.
  - Deduplicate the source first, for example with `QUALIFY ROW_NUMBER() OVER (PARTITION BY key ORDER BY updated_at DESC, <tie-breaker>) = 1`.
- **VARIANT and JSON:**
  - A missing key returns SQL NULL. A JSON `null` is a VARIANT null, which is different: `v:field IS NULL` is FALSE for it. Casting with `::string` or `TO_VARCHAR` turns it into SQL NULL, and `IS_NULL_VALUE()` tests for it.
  - Always cast extracted values (`v:amount::NUMBER(12,2)`, `v:name::string`). Uncast strings keep their JSON quotes in comparisons and output.
  - `FLATTEN` without `OUTER => TRUE` drops parent rows whose array is empty or missing.
- **Tasks are created suspended.** A new or recreated task never runs until someone runs `ALTER TASK ... RESUME`, and the root task of a task graph must be suspended before its child tasks can be changed. Check deployment scripts for the resume step.
- **Snowflake Scripting:** inside a stored procedure or block, a variable used in a SQL statement needs a colon (`WHERE id = :v_id`). Without one, Snowflake treats the name as a column identifier.

## Python connector and Snowpark
- **SQL injection:** SQL built with f-strings, `%` or `.format()` from values that come from outside the code (parameters, config, user input, file names). Use bind parameters (`cursor.execute(sql, params)`). Object names can't be bound as values, so use `IDENTIFIER(%s)` or check them against an allowlist.
- **Credentials:** hardcoded passwords, tokens, private keys or account details in code, notebooks or committed config. They should come from environment variables, a secrets manager or key-pair auth.
- **Connection strings leaking:** connection URLs logged or written to run records. Masking `user:password@` misses credentials in query parameters (`?password=`, `odbc_connect=...PWD=...`, `private_key_file_pwd=`). Also check config dumps that include the URL. See "Secrets and run records" in checklist-orchestration.md.
- **Re-querying within one run:** the same query run again later in the same job (for a report or a check) against tables that may have received new data in between. See "Consistency within a run" in checklist-orchestration.md.
- **Types coming back into pandas:**
  - An integer column (`NUMBER` with scale 0) that contains a NULL arrives as `float64`. IDs above 2^53 lose precision, and IDs stop matching their integer versions.
  - `NUMBER` columns with scale > 0 arrive as `float64`, so money columns lose decimal exactness.
  - `DATE` columns arrive as `object` columns holding `datetime.date` values. Comparing them with `pd.Timestamp` values or with strings can fail or silently match nothing.
- **Timezone-aware data with `write_pandas`:** loading tz-aware datetimes can produce shifted or invalid timestamps unless `use_logical_type=True` is passed. Read the values back to check.
- **`write_pandas` quotes identifiers by default** (`quote_identifiers=True`). Lowercase pandas column names then become case-sensitive lowercase Snowflake columns, which other SQL can't reference without double quotes. With `auto_create_table=True`, the table and its column types are also inferred from the DataFrame.
- **Connections and cursors** that aren't closed. Use a context manager or `try/finally`.
- **Large results:** `fetch_pandas_all()`, `pd.read_sql` or Snowpark `to_pandas()` / `collect()` on large results load everything into client memory. Prefer pushing the filtering or aggregation into SQL, or use `fetch_pandas_batches()`.
- **Snowpark DataFrames are lazy.** Nothing runs until an action (`collect`, `to_pandas`, `save_as_table`, `count`). Each action reruns the whole query plan unless the result is cached (`cache_result()`). Python side effects placed between transformations don't run in the order they appear.

## Transactions and reruns
- **Partial loads:** with autocommit on, a multi-statement load that fails halfway leaves partial data that looks complete. Is there an explicit transaction, or a staging-then-swap design (`ALTER TABLE ... SWAP WITH`, `INSERT OVERWRITE`)?
- **Rerun safety:** does running the same job twice for the same period duplicate data (a plain `INSERT` or `append`)? An idempotent design uses `MERGE` on a real unique key, `DELETE` + `INSERT` of the partition inside one transaction, or `INSERT OVERWRITE`.
- **Backfills and late data:** does a rerun for a past date use that date, or `CURRENT_DATE()`? A hardcoded "today" makes backfills silently recompute the wrong period.

## dbt
- **Incremental filters:**
  - `where updated_at > (select max(updated_at) from {{ this }})` skips rows with a timestamp equal to the current maximum, and permanently misses late-arriving rows. Is there a lookback window, together with a `unique_key` that deduplicates the overlap?
  - The `is_incremental()` branch must select the same columns and grain as the full-refresh branch.
- **`unique_key` must actually be unique.** With `merge`, duplicates in the new batch cause errors (see MERGE above). With `append`, every rerun duplicates rows. Check that a `unique` test backs it up.
- **Hardcoded names:** table names written out in full instead of `ref()` / `source()` break lineage, and can point production models at dev data, or the reverse.
- **Tests as evidence:** use `schema.yml` tests (`unique`, `not_null`, `relationships`, `accepted_values`) when verifying a finding. A model whose key has a `unique` test passing in CI doesn't produce join fan-out on that key.

## Cost (Low severity, report only with evidence that it matters)
- One query per ID inside a Python loop instead of one set-based query
- Functions on filter columns (`WHERE DATE(ts) = ...`, `WHERE TO_CHAR(ts, ...) = ...`) that prevent partition pruning, on a table known to be large. Filter on the raw column with a range instead.
- `SELECT *` from wide tables in scheduled jobs. Snowflake stores data by column, so unused columns are still scanned and paid for.
- Accidental cross joins, or non-equi join conditions that make row counts explode
