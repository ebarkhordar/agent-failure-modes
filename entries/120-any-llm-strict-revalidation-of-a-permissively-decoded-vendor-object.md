# A wrapper re-validates the object its vendor SDK decoded permissively, so the SDK's forward compatibility with new enum values becomes the wrapper's crash

- **Repo:** mozilla-ai/any-llm
- **Surface:** `src/any_llm/providers/openai/base.py:139`
  (`ChatCompletionChunk.model_validate`, the streaming path) and
  `src/any_llm/providers/openai/utils.py:83`
  (`ChatCompletion.model_validate`, reached through `_convert_chat_completion`), against the
  subclasses declared in `src/any_llm/types/completion.py`
- **Class:** message-conversion boundaries
- **Report:** [issue #1200](https://github.com/mozilla-ai/any-llm/issues/1200), filed by
  another user, open. Nothing was posted by us and no PR was opened: an open PR from an
  outside contributor, [#1204](https://github.com/mozilla-ai/any-llm/pull/1204), implements a
  fix. This entry records the reproduction and what it showed beyond the report. See
  **Update 2026-08-04**, which corrects what that PR is going to land and qualifies one claim
  below.

## Root cause

z.ai's GLM-5 returns `finish_reason: "model_context_window_exceeded"`. That string is not a
member of the `Literal` the OpenAI schema declares for the field, and a pydantic
`ValidationError` reaches the caller. The interesting part is where it comes from, because it
is not the OpenAI SDK.

Measured in a container with a local server standing in for the z.ai endpoint, on
`openai==2.52.0` and `pydantic==2.13.4`:

```
OpenAI SDK, streaming path      -> finish_reason='model_context_window_exceeded', no exception
OpenAI SDK, non-streaming path  -> finish_reason='model_context_window_exceeded', no exception
any-llm, both paths             -> pydantic ValidationError
```

The SDK decodes the wire payload without enforcing the `Literal`, so an unrecognised stop
reason travels through it intact. any-llm then takes that already-constructed object, dumps
it, and validates it into its own subclass. `ChatCompletionChunk` and `ChatCompletion` in
`types/completion.py` derive from the SDK's types and add fields, and `model_validate` is a
full validation pass, so the constraint the SDK never enforced is enforced here. The
narrowing is not declared anywhere in this repository: the `Literal` is inherited, and a
history probe over the fault file finds refactors and migrations but no commit that ever
restricted `finish_reason` on purpose.

Two things the report does not cover came out of the reproduction. The issue is written
against streaming and `ChatCompletionChunk`; the non-streaming path fails identically through
`_convert_chat_completion`, and the reason both break is that both re-validate. And the blast
radius is not z.ai. Walking the provider package with `ast`, 33 classes inherit
`BaseOpenAIProvider` transitively and 30 of them inherit `_convert_completion_chunk_response`
unchanged, so any of those providers emitting any stop reason outside the enum reaches the
same line. PR #1204 covers both paths, and it does so because it patches
`_normalize_openai_dict_response`, the normalizer both call chains share, rather than either
call site.

## Invariant violated

**Validating an object a library already built is not a no-op, and it is the point where a
permissive decoder is converted into a strict one.** The SDK's leniency here is a deliberate
property: providers add stop reasons, so a client that rejects unknown ones breaks every time
a vendor ships something new, and decoding without enforcing the enum is how the SDK stays
forward compatible. Re-validation discards that decision. Nothing in the wrapper announces a
stricter contract than the SDK's, and by construction it cannot be looser: whatever the SDK
tolerated, the second pass gets to reject. A conversion layer that means to be a passthrough
must either construct without validating or accept that it is now the strictest component in
the stack.

**Subclassing a vendor model to add a field is additive in the type system and multiplicative
in validation.** `class ChatCompletionChunk(OpenAIChatCompletionChunk)` reads as "the same
thing, plus ours". What it actually creates is a second model with every parent constraint
attached, and a reason to run validation again in order to produce it. The cost is invisible
at the declaration site and lands at the call site, and the constraint that fires was written
by someone else, in another repository, for a schema the wrapper does not own.

**A closed enum over values chosen by a remote server is a liability wearing the costume of a
safety property.** `finish_reason` is data from a third party. Its value set is open in
practice and closed in the type, so the type is a prediction about other companies' roadmaps,
and it is wrong on a schedule. The cost falls hardest on exactly this kind of library: a
normalization layer exists because providers differ, so it will meet every novel value first,
and it is the component least able to afford rejecting them. Where a field is an extension
point for the remote side, the parsing layer should widen and map, and let a narrow type live
further in, past the point where unknown values can be handled rather than raised on.

**A report is shaped by the reporter's call site, so the path they name is evidence about how
they use the library and not about the extent of the defect.** One user streams, so the issue
is about streaming. The same fault sat in the non-streaming path the whole time, reachable by
a different call chain through a different function, which is why reading the reported path
carefully does not find it. Locating the shared ancestor of both paths is what sizes the bug,
and it is also what makes a one-place fix possible.

## Trigger

Any provider returning a `finish_reason` outside the OpenAI enum, on either the streaming or
the non-streaming path. z.ai GLM-5 returns `model_context_window_exceeded` when the context
window is exceeded, so the failure lands precisely on the request that already went wrong,
and it converts a recoverable condition the caller could have inspected into an exception
from the parsing layer. No configuration avoids it: the re-validation is on the default path
for the 30 providers that do not override the chunk converter.

## Repro

Clean `python:3.12-slim` container at `d277097b`, still `main` at the time of writing,
installing the package from a full-history clone with the `zai` and `openai` extras. A local
`http.server` returns a canned OpenAI-shaped payload carrying the offending
`finish_reason`, so no credentials and no outbound network are involved. The same payload is
fed first to the OpenAI SDK directly and then through any-llm, on both the streaming and the
non-streaming path, which is what separates "the value breaks parsing" from "the value breaks
this wrapper's second parse". Provider counts come from an `ast` walk of
`src/any_llm/providers`, resolving base classes transitively, rather than from a text search.

The history was read on an unshallowed clone with the probe strings escaped for regex
metacharacters, and cross-checked with `grep -cF` so that a zero-hit probe would have been
read as broken rather than as an absence of history. The raw probe did return zero and the
escaped one returned five, all of them refactors and migrations touching the fault file, none
of them narrowing the field.

**Not verified:** no request was made to a real z.ai endpoint, so the exact wire payload is
modelled on the one in the report rather than captured. The claim that PR #1204 resolves both
paths comes from reading its diff and its tests, not from running them.

Verified 2026-08-01 at `d277097b`. Reported upstream by another user; the fix is open and not
merged, so both re-validation sites still ship as described.

## Update 2026-08-04

Two things in the review of #1204 correct this entry, and both were caught by people upstream
rather than by us.

**The fix is moving out of the shared normalizer, so the blast radius measured above stays
open.** The maintainer asked why the change was in the OpenAI provider rather than in z.ai's,
and the author agreed to move it to a helper in `providers/zai/utils.py`, applied as a pre
pass on `_convert_completion_response` and `_convert_completion_chunk_response`. That is the
right call for the repository: the value is one vendor's, and putting a vendor's vocabulary in
the shared OpenAI path makes every other provider inherit a mapping that is not true of them.
It does mean the sentence above, that locating the shared ancestor "is also what makes a
one-place fix possible", describes a fix that will not land in that form. What lands covers
z.ai on both paths. The other 30 providers inheriting `_convert_completion_chunk_response`
keep the re-validation exactly as described, so the general defect this entry is about is not
what #1204 resolves.

**The mapping is an approximation, not an equivalence, and the enum is short by three rather
than by one.** The author found that z.ai documents the value. Read directly on 2026-08-04,
z.ai's Chat Completion reference describes `choices[].finish_reason` as "Reason for model
inference termination. Can be `stop`, `tool_calls`, `length`, `sensitive`,
`model_context_window_exceeded` or `network_error`." So `length` and
`model_context_window_exceeded` are two distinct documented outcomes on the vendor's side, and
folding one onto the other discards a distinction the vendor draws. It also means `sensitive`
and `network_error` sit outside the OpenAI literal set too: three of z.ai's six documented
finish reasons fail the re-validation today, and #1200 is the one of the three that somebody
happened to hit.

That count is the strongest evidence in this entry for its own third invariant, and the entry
did not have it. "A closed enum over values chosen by a remote server is a liability wearing
the costume of a safety property" was argued from the mechanism; the measurement is that a
single vendor's published list already falls half outside the enum, before any roadmap moves.
It also sharpens the prescription. "The parsing layer should widen and map" is too quick,
because `sensitive` and `network_error` have no honest OpenAI equivalent and mapping them
anywhere would report a censored or failed generation as a normal stop. Widening is the part
that is always right; mapping is only right where the target value is true, and where it is
not, the layer needs somewhere to put a value it can carry without either lying about it or
raising.
