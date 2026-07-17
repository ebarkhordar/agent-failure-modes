# One of two consumers of the same header parses it, the other crashes on it

- **Repo:** IBM/mcp-context-forge
- **Surface:** `mcpgateway/utils/retry_manager.py::ResilientHttpClient` (`request`, `stream`)
- **Class:** error handling & success reporting
- **Report:** [issue #5674](https://github.com/IBM/mcp-context-forge/issues/5674)
  (deterministic repro, fix offered)

## Root cause

`ResilientHttpClient` reads the `Retry-After` header on an HTTP 429 in two
places. RFC 9110 section 10.2.3 defines `Retry-After` as either `delay-seconds`
or an `HTTP-date`, and real servers and CDNs send both.

`stream()` parses it defensively: the `float(retry_after)` is wrapped in
`except ValueError`, so an `HTTP-date` value that is not a number falls through
to the normal backoff and the retry proceeds. `request()` does the same
`float(retry_after)` with no guard. On an `HTTP-date` it raises `ValueError`, and
`_should_retry(ValueError, None)` returns False, so instead of retrying the 429
the client raises the parse error to the caller. A response the client was built
to handle becomes a hard crash, on the more commonly used of its two methods.

A second gap sits alongside the first: neither 429 block clamps the resulting
sleep to `self.max_delay`, though the client's own `_sleep_with_jitter` does. A
server-controlled `delay-seconds` is honored unbounded.

## Invariant violated

Two code paths that consume the same external grammar must implement the same
parse. Hardening one does not protect the other, and the two drift precisely
because each reads as correct on its own: the defensive `try` in `stream()` gives
no hint that `request()` lacks it. When a client library already contains the
correct handling once, the bug is not a missing idea, it is one call site that
fell out of an existing convention, and the diff between the two is free to find.

The narrower rule: a retry path must not crash on an input the spec it implements
explicitly permits. A well-formed `Retry-After` the client exists to honor should
never turn a retryable 429 into a fatal error. And any sleep derived from a
server-controlled header has to be clamped to the client's own ceiling, because
the header is attacker-influenced input, not a trusted constant.

## Trigger

Any upstream that answers a request routed through `request()` with an HTTP 429
carrying an `HTTP-date` `Retry-After`. This is RFC-legal and emitted by real
infrastructure. The same response through `stream()` retries cleanly, so the
failure depends only on which method the caller used.

## Repro

Clean `python:3.12-slim` container, `pip install -e .` at `5eece0fc`, driving a
real local `http.server` that returns 429 with an `HTTP-date` `Retry-After`, no
mocking on the request path, `__file__` provenance confirmed at
`/src/mcpgateway/utils/retry_manager.py`. `request()` raised `ValueError`;
`stream()` against the identical response retried and returned gracefully.

Reported as an issue with a PR offered: the repo's CONTRIBUTING mandates an
issue-first triage gate before an implementation PR.
