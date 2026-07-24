# A "50th percentile" index keeps a `- 1` left over from a 25th-percentile version, so for an odd sample it returns below the median (the minimum at n=3)

- **Repo:** LearningCircuit/local-deep-research
- **Surface:** `src/local_deep_research/web_search_engines/rate_limiting/tracker.py`,
  `AdaptiveRateLimitTracker._update_estimate` (the `percentile_50` index)
- **Class:** indexing, ordering & counting contracts
- **Fix:** [PR #5201](https://github.com/LearningCircuit/local-deep-research/pull/5201)
  (in review; self-discovered, no issue)

## Root cause

`_update_estimate` sets its base wait to what its own comment calls the "50th
percentile (median) of successful waits", indexing the sorted list as:

```python
percentile_50 = successful_waits[
    max(0, int(len(successful_waits) * 0.50) - 1)
]
```

The trailing `- 1` is a remnant of an earlier 25th-percentile implementation: the
factor was later swapped from `0.25` to `0.50` (in the commit whose message is "use
median instead of 25th percentile") but the `- 1` was kept. For an odd count that
index lands one slot below the true median, and at `n == 3` it returns index `0`,
the fastest observed wait rather than the middle one. `n == 3` is not a corner: the
method returns early below three attempts, so the first estimate update typically
runs on exactly three samples. For an even count the same expression already
returned the lower of the two middle waits, a valid lower-median pick, so the
defect is confined to odd `n`.

The learned base is systematically shorter than the estimator's own stated target,
so on the common early-life path the limiter anchors to the minimum instead of the
median.

## Invariant violated

The median of a sorted list of length `n` is at index `(n - 1) // 2` (lower median),
not `int(n * 0.50) - 1`. The two formulas agree for even `n` but diverge for odd
`n`, where the second sits a full slot low and collapses to index `0` at the
smallest `n` the code ever evaluates. When a percentile index is edited by changing
the multiplier, the offset that was correct for the old percentile does not carry
over: an index expression is a single contract between the factor and the `- 1`, and
touching one term without the other leaves a formula that still runs, still returns
an in-range element, and still looks like "about the middle", while pointing at the
wrong order statistic on exactly the sample sizes the guard admits first. A comment
that names the intended statistic ("median") is the specification; an index that
does not compute it is the bug, even though every value it returns is a real member
of the list.

## Trigger

Any first estimate update, which runs on the minimum qualifying sample of three
successful waits: the old index returns the fastest of the three instead of the
median. More generally, any odd number of successful waits yields a below-median
base. Even counts are unaffected. The effect is bounded (0.01 to 10s) and
EMA-smoothed, so it biases the limiter toward waiting less than its own 50th
percentile rather than failing outright.

## Repro

Docker-verified this session in a clean `python:3.12` container (editable install,
`tracker.__file__` provenance confirmed against the checkout). Because the first
estimate has no prior value there is no EMA blend, so the base equals the raw
indexed element: for `[0.5, 1.0, 1.5]` the old index returns `0.5` (the minimum)
where the median is `1.0`; for `[1.0, 2.0, 3.0, 4.0, 5.0]` it returns `2.0` where
the median is `3.0`; for seven samples `3.0` where the median is `4.0`. Even counts
match before and after. The fix uses `(len(successful_waits) - 1) // 2`, which is
the median for odd `n` and the unchanged lower-median for even `n`, so the
repo's pre-existing even-`n` assertions (`test_all_successes_uses_median`,
`test_two_successes_median_index`) stay green rather than being rewritten to a new
convention; the three tracker test files pass (129) and `ruff check` /
`ruff format --check` are clean. Not measured: the downstream retry-timing effect
follows from how the base feeds `get_wait_time` and was not exercised end to end.
