# `"error" in result` is a key test on a dict, a substring test on a string, and a crash on anything else

- **Repo:** IBM/mcp-context-forge
- **Surface:** the non-2xx error branch of `invoke_tool` at
  `mcpgateway/services/tool_service.py:5628-5644`, with the same unguarded membership test in
  the non-standard-2xx branch at `:5653-5659`. The `response_text` key it falls back to is
  written only by `_handle_json_parse_error` at `:635`
- **Class:** error handling & success reporting
- **Report:** reproduced publicly on
  [issue #6027](https://github.com/IBM/mcp-context-forge/issues/6027#issuecomment-5148586790)
  (open, filed by another user against a live Cloudflare tool). No PR from us: the repo
  deprioritises outside implementation on this file, and the remedy spans two branches whose
  key set is a maintainer's call.

## Root cause

When an upstream REST tool answers non-2xx, the gateway parses the body and tries to lift a
human-readable message out of it. It recognises one spelling. `result` is whatever
`response.json()` returned, and the branch asks `"error" in result` before reading
`result["error"]`, with a fallback that renders the status line.

Two things go wrong, and only the first was reported.

The recognised key set is one name wide. Driving the real `invoke_tool` against a mocked 403
with nine different bodies:

```
[error (singular)]                   content='Invalid zone identifier'
[errors (Cloudflare/json:api)]       content='HTTP 403'
[message]                            content='HTTP 403'
[detail (FastAPI)]                   content='HTTP 403'
[array]                              content='HTTP 403'
[bare string containing "error"]     RAISED ToolInvocationError: string indices must be integers, not 'str'
[bare string without it]             content='HTTP 403'
[null]                               RAISED ToolInvocationError: argument of type 'NoneType' is not iterable
[number]                             RAISED ToolInvocationError: argument of type 'int' is not iterable
```

Four of the five dict-shaped bodies carry a perfectly good diagnostic and reach the caller as
`HTTP 403`. None of the four is exotic: `errors[]` is what json:api and Cloudflare emit,
`detail` is FastAPI's own default error envelope, and `message` is ubiquitous. The branch really
is two-way for a body that parses as JSON, because `response_text` has exactly one assignment
site in the package and it is on the parse-failure path.

The three `RAISED` rows are the second failure mode and are not in the report. `response.json()`
is not obliged to return a dict, and `in` does not mean the same thing on the alternatives. On a
`str` it is a substring test, so any JSON string body containing the word "error" takes the
`result["error"]` branch and dies one line later on the subscript. On `None` or a number the
membership test itself raises. In all three cases the caller receives a `ToolInvocationError`
instead of a result with `isError: true`, which is a different and more disruptive failure for
an agent than the one filed: a tool that reports an error is handled, a tool that raises usually
is not.

## Invariant violated

**`in` is defined on nearly every container type and asks a different question on each, so a
membership test against a value the peer chose is not one test, it is a dispatch table you did
not write.** Key on a dict, substring on a string, element on a list, exception on `None` and on
numbers. Nothing in the expression records which of those was intended, so the reader's eye
supplies the dict and moves on. The substring case is the nastiest of the four because it does
not fail at the predicate: it takes the branch, dies at `result["error"]`, and produces
`string indices must be integers`, a message that reads like malformed input rather than like a
wrong guard. The traceback points one line past the defect and names the wrong party.

The general form is that **`response.json()` is typed "whatever arrived", and every operation
applied to it before an `isinstance` check inherits that.** A parsed body is one of the few
values in a program whose type is chosen by a stranger; treating it as a dict because it usually
is a dict is the same class of assumption as trusting its contents, and it fails on the same
inputs an attacker or a misconfigured proxy would supply.

**An error normaliser that recognises one spelling of "the message" discards the diagnostic for
every peer that spells it differently, and it fails toward output that looks like a complete
answer.** `HTTP 403` is not a report, it is the status line restated. It is well formed, it is
not empty, and nothing downstream can distinguish it from a server that genuinely sent no body.
An operator reading it concludes the upstream said nothing, which is precisely the wrong next
step when the upstream said "Invalid zone identifier". A normaliser that dropped the field
loudly would be a smaller bug than one that substitutes a plausible sentence.

**All of this lives on the error path, which is the one region of a program with no successful
outcome anywhere to contradict it.** The normaliser only runs when something has already gone
wrong, so every observation of its behaviour arrives pre-attributed to the anomaly that
triggered it: the user who sees `HTTP 403` blames their credentials, and the user whose agent
gets a `ToolInvocationError` blames the flaky upstream. Measured here, no unit test pins the
bare `HTTP {status}` fallback, and the history shows the key set was introduced with the initial
open-source import and never widened and never reverted, so nothing about the single key is
load-bearing. It is not a decision that was made and defended. It is a decision nobody had a
reason to revisit, which is what an error path buys you.

## Trigger

Any REST-integration tool whose upstream returns a non-2xx status with a JSON body that is not a
dict containing the key `error`. That covers the data loss for four common envelope shapes and
the crash for three body types. No Cloudflare account and no credentials are needed; the
reporter's path required a live tool, and this one does not.

## Repro

Clean Python 3.12 container, `pip install -e .` at `main`
`a48826ba0b07bd7b2a33123d67396a39dc23289d`, running a parametrised test placed in the repo's own
`tests/unit/mcpgateway/services/` tree so it can reuse the existing `tool_service`, `mock_tool`
and `mock_global_config_obj` fixtures. The test builds a 403 response per body shape, points
`tool_service._http_client.get` at it, calls the real
`invoke_tool(test_db, "test_tool", {}, request_headers=None)`, and prints either the returned
content or the exception. The full source is in the linked comment; its output is the nine-row
table above.

The `response_text` claim was checked by enumeration rather than by reading the branch: that key
has one assignment in the package, in `_handle_json_parse_error`, which is why a body that
parses successfully can never reach the richer fallback.

The history gate ran on an unshallowed clone:
`git log -G '"error" in result' --follow -- mcpgateway/services/tool_service.py` traces the test
to the initial open-source import, with the current three-way form arriving in `2b6a68ad5`
(PR #3873, "handle non-JSON responses and query params in REST tools"), whose subject is the
non-JSON fallback and not the key set. No commit widens the keys and none reverts such a
widening.

**Not verified:** the transport is mocked at `_http_client.get`, so this is a statement about
what `tool_service` does with a response, not an observation of what Cloudflare returns. The
second site at `:5653-5659` was read, not executed; the claim there is that it carries the same
unguarded membership test, not that its behaviour was measured.

Verified 2026-08-01 at `main` `a48826ba0b`, which is still the tip as of this entry. Reported,
not fixed upstream at the time of writing.
