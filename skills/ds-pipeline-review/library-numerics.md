# Library code, refactors and numerical edge cases

Use this checklist for library or public-API code, and for any change described as a refactor, a vectorization, a performance improvement or a port of another implementation (for example "match R's ..."). These changes promise identical behavior, so any difference is a bug unless the PR says otherwise.

## Differential testing (do this, don't just reason about it)
Copy the old and the new version of the changed function into a standalone script and run both on a grid of inputs. Compare:
- the values
- the output dtype (including resolution and timezone)
- exceptions
- warnings: a new `FutureWarning` or `DeprecationWarning` is user-visible, and breaks users who run with warnings as errors

Build the grid from **every category of input the public API accepts**, not from typical inputs. For each argument, include:
- **Types and dtypes:** int8/int32/int64, unsigned, nullable `Int64`, pyarrow types, float, object; datetime resolutions `s`/`ms`/`us`/`ns`, tz-aware, pyarrow timestamps and dates; Series vs Index vs ndarray, with a non-default index
- **Kinds of parameters:** for offsets and frequencies, fixed (`Hour`, `Day`) and calendar offsets (`MonthBegin`, `BusinessDay`), plus `DateOffset` with sub-unit components (`milliseconds=500` on second-resolution data)
- **Boundary values:** 0, 1, negative, the smallest valid value (for example `period=1` or `period=2`), and values near the dtype's limits (very large `n`, steps that overflow int8 or int64)
- **Shapes:** empty, a single element, all equal, already sorted vs unsorted, NaN or NaT present, all NaN

Report any difference the PR didn't announce. In particular, report:
- **A raised error that became a silent wrong value** (overflow that now wraps instead of raising, a narrowing cast that now truncates). This is worse than the old behavior, even though the old behavior was an error.
- **A value that became an error** for input the old code accepted, such as a boundary parameter like `period=1`.
- **Precision lost** by casting a result back to the input dtype or resolution.

When you report "No issues found" for such a change, list the grid you actually ran, so the reader can see what was and wasn't covered.

## Numerical pitfalls to look for
- **Integer arithmetic:** multiplying or adding in fixed-width integer types without overflow checks (`n * step` in int64, `arange(...) * freq` cast to a narrow dtype). NumPy wraps silently, while pandas and Python ints raise.
- **Casting back to the input dtype:** `out.astype(input.dtype)` at the end of a function can truncate resolution (milliseconds dropped from `datetime64[s]`), wrap integers, or drop timezone information.
- **Absolute tolerances on scale-dependent quantities:** a fixed `1e-10` threshold on a variance or difference behaves differently for series measured in different units. Report it when the change introduces or moves such a threshold.
- **Parity with a reference implementation (R, statsmodels, a paper):** check boundary cases, missing-value handling, ddof, and window or default parameters against the reference. Also check the docs and release notes don't claim more parity than the code delivers.
- **Removed public names:** removing or renaming anything importable from a public module without a deprecation. Underscore-prefixed names re-exported from a package `__init__` count as semi-public.
- **Dependency ranges:** behavior that only works on part of the declared version range (pandas, numpy, statsmodels). Check the minimum supported version when the code uses newer behavior.
