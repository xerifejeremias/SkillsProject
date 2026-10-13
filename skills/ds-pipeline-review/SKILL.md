---
name: ds-pipeline-review
description: Reviews Python data science pipeline code for correctness bugs, focusing on pandas, Polars, SQL, Snowflake and dbt (rows silently dropped or duplicated, NULL/NaN handling, join fan-out, dtype and timezone errors, metric and aggregation mistakes, data leakage, forecasting, backtesting, prediction-interval and model-selection mistakes, statsmodels/ARIMA and neural forecasting model bugs, Snowflake connector pitfalls, non-idempotent loads, scheduled jobs that fail silently). Use when the user asks to review, check or audit a data pipeline, ETL/ELT job, feature engineering or model training code, pandas transformations, SQL queries, Snowflake or dbt code, or a diff/PR/branch that touches them.
argument-hint: "[files | folder | PR number | branch]"
---

# Data science pipeline review

Review Python, pandas, SQL, Snowflake and dbt code the way a senior data engineer would. The top priority is bugs that produce **wrong data without crashing**: rows silently dropped or duplicated, NULLs miscounted, timestamps shifted, metrics computed on the wrong grain, or leakage into training data. Pipelines rarely fail loudly. They make the numbers wrong.

Silent data bugs are not the only target. Also report:
- crashes and confusing errors on realistic input or configuration, especially regressions where something that used to work now fails
- pipelines that silently **don't run** or don't report a failure (scheduling, alerting, clean-up). Nobody notices a job that didn't run, just as nobody notices a wrong number.

Rank these below silent wrong data, but don't drop them because they fail loudly.

**High signal only.** A short list of real bugs is worth more than a long list of suspicions. If you aren't sure an issue is real, don't report it. Reporting nothing is a valid outcome.

## 1. Work out what to review

- Target: $ARGUMENTS
- If a target is given above, review it: files, a folder, a PR (`gh pr diff <n>`, `gh pr view <n>`) or a branch (`git diff <main-branch>...<branch>`).
- Otherwise review the current changes: `git diff HEAD` (staged and unstaged), or `git diff <main-branch>...HEAD` if the working tree is clean.
- If there is no git repository and no target, ask what to review.

Then gather context:
- **Intent:** the PR title and description, or the commit messages. You need the intended behavior before you can say something is wrong.
- **Project rules:** the root `CLAUDE.md` and any `CLAUDE.md` in the folders of the changed files, plus decision or design docs they point to. Violating a rule in them counts as a finding only when you can quote the exact rule and the file falls under its scope. A quoted project rule takes precedence over a code comment or docstring that calls the behavior intended. For example, if the rules say "fail loudly", a commented silent fallback is still a finding.
- **Kind of code:** is this a library or public API, or a single pipeline? For a library, any input the public API accepts can happen (user columns with common names like `month` or `date`, numeric-looking IDs, NaNs in any column, unsigned or narrow integer dtypes, negative steps, boundary parameters), so don't dismiss such inputs as unrealistic. Only the API's documentation or validation can rule an input out. For a single pipeline, judge realism against its actual sources.
- **Versions:** check `pyproject.toml`, `requirements*.txt` or the lock file for the pandas, Snowflake connector, Snowpark and dbt versions. Several pitfalls depend on version, especially pandas 2.x vs 3.x. Only report version-specific issues that apply to the version actually in use. If nothing pins the versions (no lock file or upper bounds), the code will run on whatever is current. A breakage on current versions (a removed API, numpy refusing ragged arrays, a pandas argument conflict) then deserves at least an "Also noticed" line.
- **Linters:** note which linters the project configures (ruff, flake8, pylint, sqlfluff) and which rules they enable. Don't report issues those tools already catch, and don't run them to check.

## 2. Read the code and follow the data

Read each changed file in full, not just the diff hunks. Pipeline bugs usually come from how a change interacts with the code around it: a join added here duplicates rows that are summed three functions later. For each DataFrame or table touched by the change, work out:
- **Where it comes from:** its source, its grain (what one row represents) and which keys are unique
- **Where it goes:** the next transformation, and the final output (table, report, model features, file)

Read callers, upstream queries, schemas, dbt `schema.yml` tests and existing tests when you need them to prove or disprove a bug.

SQL counts as code too: check `.sql` files, SQL strings embedded in Python, dbt models and Snowflake scripts. Notebooks are in scope when they produce something used downstream (a scheduled job, an exported table, a trained model). For purely exploratory notebooks, report only bugs that would make the conclusions wrong.

Three checks catch most regressions in pipeline changes:

- **Old vs new behavior:** when the diff changes a filter, join, condition, default, date boundary or aggregation, compare the old and new behavior explicitly. Many pipeline regressions are one-character changes (`<` to `<=`, `inner` to `left`).
- **Replaced code:** when a function, reader or step is replaced (a new loader instead of an old one, a new selection rule, a rewritten query), list what the old code handled: input formats such as integer `YYYYMMDD` dates or numeric-looking IDs, dtypes, empty input, missing columns, and options that were passed. Then check, or test with a few rows, that the new code still handles each of them.
- **Other readers of the same data:** when a change widens what the data can look like (new ID formats, renamed or mapped columns, new dtypes, a new source such as SQL instead of CSV), search for every other place that reads, writes or joins the same data. Examples are secondary input files, saved models, outputs, reports and other pipelines. A change applied to the main input but not to a side file is a classic bug. List the readers you find, then check each one against every dimension of the change, one by one:
  - **column names:** does it apply the same renaming or mapping?
  - **dtypes:** does it read keys and dates with the same types (for example `dtype={id: str}`)?
  - **formats:** does it accept the newly allowed values (ID formats, date formats, units)?
  - **source:** does it read from the same place, or does it still assume the old one (a hard-coded CSV path when the input can now be SQL)?

  Checking one dimension per reader and stopping there is how half of these bugs get missed.

- **New or changed parameters:** when the change adds or changes a parameter, find every public entry point that accepts or documents it, including wrappers and convenience functions that call the changed function (for example `backtest()` and `residuals()` calling `historical_forecasts()`). Check that each one actually forwards it. A documented parameter that a wrapper accepts and silently ignores is a bug.

A code path that becomes reachable through the change counts as new code, even when the lines themselves didn't change.

**"Pre-existing" has to be proven, not assumed.** Before dropping a candidate as pre-existing, trace or run the same scenario on the old code and confirm the behavior was identical. Code whose lines didn't change can still break when the change alters the state it depends on: what an attribute holds after a call, which object is truncated or copied, or the order in which methods run. If the old and new behavior differ, the change caused it. If you can't tell, report it as plausible and explain why, rather than dropping it. For example, a report step that re-reads the input was harmless with a fixed CSV file, but it re-queries live data once the input can come from SQL.

## 3. Look for bugs

Go through the checklists that apply to the code in front of you. Read only the ones you need:

- [pandas.md](pandas.md): pandas, Polars, and the Python around them
- [sql-snowflake.md](sql-snowflake.md): SQL, Snowflake, the connector, Snowpark and dbt
- [analytics-ml.md](analytics-ml.md): metric logic, model selection, leakage in feature engineering, modelling and forecasting, classical statistical models, prediction intervals, and neural network model code
- [orchestration.md](orchestration.md): scheduling, alerting, clean-up, run records and secrets in logs. Use it whenever the change touches how jobs are run or monitored.
- [library-numerics.md](library-numerics.md): differential testing and numerical edge cases. Use it for library or public-API code, and for any refactor, vectorization, performance change or port of a reference implementation. These changes promise identical behavior, so test that systematically: don't rely on a few typical inputs.

While reading, actively try to break the change with realistic data states: NULLs, empty frames or partitions, duplicate keys, late-arriving rows, new category values, DST transitions, month and year boundaries, and reruns of the same job.

**Then make one pass without the checklists.** Checklists cause tunnel vision. For each changed file, ask what the change breaks for the code that calls it and for the code it calls, and how a user configuring it in a reasonable way could get a wrong result or a failure. Report any real bug you find this way, even if no checklist mentions it.

For every candidate issue, write down:
- **Location:** the exact `file:line`
- **Trigger:** the concrete data state or input that causes it (for example "a customer with two rows in `addresses`")
- **Actual vs expected:** what the code produces and what it should produce (for example "their revenue is counted twice in `monthly_totals`, but it should be counted once")
- **Impact:** where the wrong result ends up (a dashboard, a model feature, a downstream table)

If you can't fill all four in concretely, it isn't a finding.

## 4. Verify before reporting

Try to prove each candidate wrong before you keep it:
- Is it already handled upstream? For example: the key is deduplicated earlier, NULLs are filtered, the column is cast, or a `validate=` or dbt `unique` test guarantees it.
- Does a constraint, schema, test or comment show the triggering data can't happen?
- Is the triggering data realistic for this source? A nullable column, an event table with no unique key or a timestamp column from a different system makes it realistic. An assumption nothing supports makes it hypothetical, so drop it.
- If a few rows of data would settle it, build them and run a quick check with pandas or SQL. Use the project's environment, so the pandas version matches.

Drop anything that doesn't survive. Mark each survivor:
- **confirmed:** you proved it, either by running a check or through an unambiguous code path
- **plausible:** it depends on data you can't see, the triggering data state is realistic, and nothing in the code rules it out

## 5. What not to report

- Style, formatting, naming, readability, or "best practice" advice that doesn't fix a bug
- Issues the project's linters already catch
- Pre-existing issues the change doesn't touch, make worse or make newly reachable. When the user asks to review whole files or a folder rather than a diff, everything in them is in scope.
- Requests for more tests or validation in general. Do report a changed test that asserts wrong behavior.
- Things that look wrong but are correct for the intended behavior. For example, an inner join is right when unmatched rows should be dropped.
- Hypothetical failures that need unrealistic data or call orders
- Performance and cost, except in the cases listed under severity below

## 6. Report

Rank findings by impact:

- **High:** wrong, lost or duplicated data reaching an output (a table, report, metric or model); a logic bug that makes model or forecast quality worse for many series or rows under the default or documented settings, for example a selection, weighting or feature rule that works against its own stated purpose; a load that fails halfway and leaves partial data that looks complete; leakage that inflates model metrics; credentials in code; SQL built from untrusted input
- **Medium:** a crash or misleading error on realistic data or configuration (an empty frame, NULLs, a new category, a documented config option); a regression where input that used to work now fails; wrong results in a narrow but reachable case; a job that isn't safe to rerun; a scheduled job that can silently not run, or fail without an alert
- **Low:** narrow bugs with a small blast radius; a silent fallback, overwrite or row drop that changes the output without any warning or error (at least Low, higher if it is common or breaks a project rule); cost or performance problems with clear evidence they matter. Examples are one query per row inside a Python loop, a full scan of a table known to be large in a scheduled job, or loading a full table into pandas only to filter it.

Use this format for each finding:

```
### <n>. <one-line summary>  [High | Medium | Low] [confirmed | plausible]
`path/to/file.py:42`

**Trigger:** <data state or input that causes it>
**What goes wrong:** <actual vs expected result, and where it ends up>
**Fix:** <the smallest change that fixes it, with a short code snippet when it helps>
```

Report each unique issue once. If the same bug appears in several places, list all the locations under one finding.

After the findings, you may add an **Also noticed** section: up to 5 one-line items. These are Low issues you saw but didn't fully verify, or borderline cases you decided not to report as findings. Give each a `file:line` and one sentence on why it might matter. Never put High or Medium suspicions here: verify them or drop them. Leave the section out if there's nothing worth a line.

End with one line saying what you reviewed (files or diff range). If nothing survived verification, say "No issues found." and name the riskiest areas you checked.

Don't change any code unless the user asks. If they ask you to fix the findings, apply the smallest fixes and re-run the project's tests if it has any.
