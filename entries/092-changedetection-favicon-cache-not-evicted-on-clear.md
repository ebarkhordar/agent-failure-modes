# A process-global filename cache is evicted only on the write path, so deleting the file leaves the name behind

- **Repo:** dgtlmoon/changedetection.io
- **Surface:** `changedetectionio/model/Watch.py`, the module-level
  `_FAVICON_FILENAME_CACHE` (declared `:49`, read `:904-905`, populated `:910`, evicted
  only at `:879`), against `clear_watch()` at `:335`
- **Class:** cache keys & invalidation
- **Report:** reproduced publicly on
  [issue #4277](https://github.com/dgtlmoon/changedetection.io/issues/4277#issuecomment-5080275142)
  (open). No PR from us: this lane already holds two open PRs on this solo-maintained
  repo, which is its per-repo ceiling, so the reproduction went out as triage with the
  fix location named and an offer to take it.

## Root cause

`Watch.get_favicon_filename()` globs the watch's data directory for `favicon.*` and
memoizes the basename it finds in a dict that lives at module scope, keyed by
`data_dir`:

```python
_FAVICON_FILENAME_CACHE: dict = {}      # Watch.py:49
```

Exactly five lines in the package touch that dict (an uncapped `grep -c`, not a
windowed read): the declaration, the memoizing read and write inside
`get_favicon_filename()`, and one `pop` inside `bump_favicon()` at `:879`.
`bump_favicon()` is the *write* path, called when a fetch has just saved a new favicon.

`clear_watch()` is the delete path. It unlinks every file in the data directory at
`:335` and then resets eleven keys on the watch, none of them favicon related. It never
pops the cache. So after clearing history the file is gone and the remembered name is
not: `get_favicon_filename()` keeps returning `favicon.png` for a directory that
contains only `watch.json`.

The stale name is then rendered as a real URL. `watch-overview.html:355` emits the
`data-src` under `{% if favicon %}` at `:400`, and `static_content` calls
`send_from_directory` on the missing file (`flask_app.py:862-872`), which is the broken
image the reporter sees. `api/Watch.py:446-454` has the same shape.

Which fetcher the watch uses decides whether the staleness is permanent or transient,
and that is the part that makes the bug look intermittent rather than deterministic.
`favicon_blob` is assigned only by `content_fetchers/playwright.py:349` and
`puppeteer.py:447`, never by `requests.py`. Under the Basic fetcher the guard at
`worker.py:634` is therefore always false, `bump_favicon()` never runs again, and the
single eviction site is never reached. Under Playwright the next successful check
rewrites the file and pops the cache, so the window closes within one check interval
and the bug hides.

The same missing eviction appears twice more: `_delete_watch()` rmtrees the directory
at `store/__init__.py:451` without popping, and `get_favicon_mime_type()` is
`lru_cache`d by path at `favicon_utils.py:9`.

## Invariant violated

A cache over a resource must be invalidated by every path that changes the resource,
not only by the path that creates it. Eviction placed next to the write is the easy
half of the contract, because the writer is the code that knows the new value; the
deleter knows only that the old value is now wrong, which is exactly the case a cache
cannot detect for itself.

The general shape is worth naming because it survives review so easily. A cache added
alongside a producer looks complete: the producer sets, the producer clears, every test
that exercises the producer passes. The counterexample is not a race or an ordering
subtlety, it is a second mutator somewhere else in the package that predates the cache
or was written without knowledge of it. So the question to ask of any memo is not "does
the writer evict" but "how many functions can make this value wrong", and that is an
enumeration over the whole package, not a reading of the caching function.

Two properties make this class of defect quiet. A module-level dict outlives every
object it describes, so the stale entry is scoped to the process rather than to the
record, and restarting the app clears it, which turns a deterministic bug into a
"sometimes" bug in every report. And the consumer of a cached *name* cannot tell a
stale answer from a fresh one, because the name is well formed either way; the error
surfaces later, in whatever code opens the file, as a missing-file symptom several
layers from the cause.

## Trigger

A watch that has a favicon saved by the Playwright or Puppeteer fetcher, is then
switched to the Basic fetcher, and has its history cleared. The favicon file is
deleted, the overview page keeps requesting it, and nothing evicts the name until the
process restarts or a browser fetch saves a favicon again.

## Repro

Clean `python:3.11-slim`, `pip install -r requirements.txt`, HEAD `fdfa3859`, no browser
needed: drive the real `ChangeDetectionStore.clear_watch_history` (the same call the UI
blueprint makes at `blueprint/ui/__init__.py:188`) after a real `bump_favicon()`, and
read `get_favicon_filename()` on both sides.

```
before: ['favicon.png', 'watch.json'] favicon.png
after:  ['watch.json']                favicon.png
evicted: None
```

The third line pops the cache entry by hand and shows the function then returns `None`,
which isolates the cache as the whole of the defect: the filesystem state is already
correct after `clear_watch_history`.

Verified 2026-07-25 at HEAD `fdfa3859`. Re-checked 2026-07-26: `fdfa3859` is still the
tip, the five-line census is unchanged, and `clear_watch()` still unlinks without
popping, so the defect is live. Not fixed upstream at the time of writing.
