# Review method: detailed rules for each review step

This file expands the workflow in the agent instructions. Use it on every review.

## Context: kind of code, project rules, versions

- **Project rules:** the root `CLAUDE.md`, any `CLAUDE.md` in the folders of the changed files, and the decision or design docs they point to. Violating a rule counts as a finding only when you can quote the exact rule and the file falls under its scope. A quoted project rule takes precedence over a code comment or docstring that calls the behavior intended. For example, if the rules say "fail loudly", a commented silent fallback is still a finding.
- **Library or public API vs single pipeline:** for a library, any input the public API accepts can happen: user columns with common names like `month` or `date`, numeric-looking IDs, NaNs in any column, unsigned or narrow integer dtypes, negative steps, boundary parameters. Don't dismiss such inputs as unrealistic. Only the API's documentation or validation can rule an input out. For a single pipeline, judge realism against its actual sources.
- **Versions:** check `pyproject.toml`, `requirements*.txt` or the lock file for the pandas, Snowflake connector, Snowpark and dbt versions. Several pitfalls depend on version, especially pandas 2.x vs 3.x. Only report version-specific issues that apply to the version in use. If nothing pins the versions (no lock file or upper bounds), the code runs on whatever is current. A breakage on current versions (a removed API, numpy refusing ragged arrays, a pandas argument conflict) then deserves at least an "Also noticed" line.
- **Linters:** note which linters the project configures (ruff, flake8, pylint, sqlfluff) and which rules they enable. Don't report issues those tools already catch.

## Following the data

Read each changed file in full, not just the diff hunks. Pipeline bugs usually come from how a change interacts with the code around it: a join added here duplicates rows that are summed three functions later. For each DataFrame or table touched by the change, work out where it comes from (source, grain, unique keys) and where it goes (next transformation, final output: table, report, model features, file). Use callers, upstream queries, schemas, dbt `schema.yml` tests and existing tests to prove or disprove a bug when the user provides them, and ask for a specific missing file when it would settle a High or Medium candidate.

SQL counts as code too: `.sql` files, SQL strings embedded in Python, dbt models and Snowflake scripts. Notebooks are in scope when they produce something used downstream (a scheduled job, an exported table, a trained model). For purely exploratory notebooks, report only bugs that would make the conclusions wrong.

## The four regression checks

- **Old vs new behavior:** when the change touches a filter, join, condition, default, date boundary or aggregation, compare the old and new behavior explicitly. Many pipeline regressions are one-character changes (`<` to `<=`, `inner` to `left`).
- **Replaced code:** when a function, reader or step is replaced (a new loader, a new selection rule, a rewritten query), list what the old code handled: input formats such as integer `YYYYMMDD` dates or numeric-looking IDs, dtypes, empty input, missing columns, options that were passed. Then check, or test with a few rows, that the new code still handles each one.
- **Other readers of the same data:** when a change widens what the data can look like (new ID formats, renamed or mapped columns, new dtypes, a new source such as SQL instead of CSV), find every other place that reads, writes or joins the same data: secondary input files, saved models, outputs, reports, other pipelines. A change applied to the main input but not to a side file is a classic bug. List the readers, then check each one against every dimension of the change, one by one:
  - **column names:** does it apply the same renaming or mapping?
  - **dtypes:** does it read keys and dates with the same types (for example `dtype={id: str}`)?
  - **formats:** does it accept the newly allowed values (ID formats, date formats, units)?
  - **source:** does it read from the same place, or still assume the old one (a hard-coded CSV path when the input can now be SQL)?
  Checking one dimension per reader and stopping there is how half of these bugs get missed.
- **New or changed parameters:** find every public entry point that accepts or documents the parameter, including wrappers and convenience functions that call the changed function (for example `backtest()` and `residuals()` calling `historical_forecasts()`). Check that each one forwards it. A documented parameter that a wrapper accepts and silently ignores is a bug.

A code path that becomes reachable through the change counts as new code, even when its lines didn't change.

## Proving "pre-existing"

Before dropping a candidate as pre-existing, trace the same scenario through the old code and confirm the behavior was identical. Unchanged lines can still break when the change alters the state they depend on: what an attribute holds after a call, which object is truncated or copied, or the order in which methods run. If old and new behavior differ, the change caused it. If you can't tell, report it as plausible and explain why, rather than dropping it. Example: a report step that re-reads the input was harmless with a fixed CSV file, but it re-queries live data once the input can come from SQL.

## Verification

Try to prove each candidate wrong before keeping it:
- Is it already handled upstream? The key is deduplicated earlier, NULLs are filtered, the column is cast, or a `validate=` or dbt `unique` test guarantees it.
- Does a constraint, schema, test or comment show the triggering data can't happen?
- Is the triggering data realistic for this source? A nullable column, an event table with no unique key, or a timestamp from a different system makes it realistic. An assumption nothing supports makes it hypothetical: drop it.
- If a few rows of data would settle it and a Python tool is available, build them and run a quick check. Only run self-contained snippets you wrote; never run code that connects to databases, APIs or file shares. Without a tool, trace the code by hand and only label a finding confirmed when the code path is unambiguous.

## What not to report

- Style, formatting, naming, readability, or "best practice" advice that doesn't fix a bug
- Issues the project's linters already catch
- Pre-existing issues the change doesn't touch, make worse or make newly reachable. When the user asks to review whole files or a folder rather than a diff, everything in them is in scope.
- Requests for more tests or validation in general. Do report a changed test that asserts wrong behavior.
- Things that look wrong but are correct for the intended behavior. For example, an inner join is right when unmatched rows should be dropped.
- Hypothetical failures that need unrealistic data or call orders
- Performance and cost, except in the cases listed under Low below

## Severity, with examples

- **High:** wrong, lost or duplicated data reaching an output (a table, report, metric or model); a logic bug that makes model or forecast quality worse for many series or rows under the default or documented settings, for example a selection, weighting or feature rule that works against its own stated purpose; a load that fails halfway and leaves partial data that looks complete; leakage that inflates model metrics; credentials in code; SQL built from untrusted input.
- **Medium:** a crash or misleading error on realistic data or configuration (an empty frame, NULLs, a new category, a documented config option); a regression where input that used to work now fails; wrong results in a narrow but reachable case; a job that isn't safe to rerun; a scheduled job that can silently not run, or fail without an alert.
- **Low:** narrow bugs with a small blast radius; a silent fallback, overwrite or row drop that changes the output without any warning or error (at least Low, higher if it is common or breaks a project rule); cost or performance problems with clear evidence they matter, such as one query per row inside a Python loop, a full scan of a table known to be large in a scheduled job, or loading a full table into pandas only to filter it.

## Also noticed

After the findings, an optional "Also noticed" section holds up to 5 one-line items: Low issues seen but not fully verified, or borderline cases not reported as findings. Give each a location and one sentence on why it might matter. Never put High or Medium suspicions there: verify them or drop them. Leave the section out if there's nothing worth a line.
