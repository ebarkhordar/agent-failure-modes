# A two-key delta meets an if/elif dispatcher, and the signature never survives

- **Repo:** strands-agents/harness-sdk
- **Surface:** `strands/models/gemini.py::GeminiModel._format_chunk` and
  `GeminiModel.stream`, against
  `strands/event_loop/streaming.py::handle_content_block_delta`
- **Class:** streaming & usage accounting
- **Fix:** [PR #3306](https://github.com/strands-agents/harness-sdk/pull/3306) (in review)

## Root cause

Gemini issues a thought signature alongside a reasoning turn, sometimes on the
part carrying the reasoning text and sometimes on a separate part with no text at
all, and the adapter is built to round-trip it: `gemini.py:401` base64-encodes the
signature on the way in, and `_format_request_content_part` reads it back out on
the next turn to send it along with the reasoning text. Both ends are correct.
Everything between them loses it, in two independent places.

`_format_chunk` builds a single `reasoningContent` delta carrying both a `text`
key and a `signature` key. The shared aggregator dispatches on key presence:

```python
if "text" in delta_content["reasoningContent"]:
    ...
elif "signature" in delta_content["reasoningContent"]:
```

The dict literal in `gemini.py` sets `text` unconditionally, so for Gemini the
first arm always wins and the `elif` is unreachable. `state["signature"]` is
never set, `handle_content_block_stop` leaves the signature out of the reasoning
block, and the read-back on the next turn does
`content["reasoningContent"]["reasoningText"].get("signature")` and gets `None`
every time. The adapter encodes a signature, throws it away in aggregation, then
looks for it again and finds nothing.

That is the first of two losses on the same field, and the second never reaches
the dispatcher at all. `stream()` gates the whole reasoning emission on
`if part.text:` (`gemini.py:570`), so when Gemini attaches the signature to a part
carrying no text of its own, `_format_chunk` is never called for that part: no
delta is built, nothing is encoded, and the `if/elif` above has nothing to
mis-dispatch. Google's thought-signature guidance documents that shape and
`google-genai` 2.13.0 carries it in its own stream tests. We did not find this. A
reviewer did, on 2026-07-21, and reproduced it against the same base commit this
entry measured (`signature deltas=[]`, the next request replaying
`thought_signature=None`). It is fixed in `aff5179`, which emits the signature
whenever a part carries `thought_signature` and is not a function call, opening a
`reasoning_content` block first if none is open.

Across every adapter that builds a `reasoningContent` delta (`anthropic`,
`bedrock`, `gemini`, `llamacpp`, `openai`, `openai_responses`), Gemini was the
only one packing both keys into one delta, which is exactly why the `if/elif`
holds everywhere else. `anthropic.py:314` emits `signature_delta` as its own
single-key delta, and `bedrock.py:1139` yields the text delta and then the
signature as a separate delta.

## Invariant violated

A dispatcher that selects among mutually exclusive arms by testing key presence
is asserting a contract: each message carries exactly one of these keys. That
contract binds every producer feeding it. It is written down nowhere, no type
expresses it, and the `if/elif` that depends on it reads as ordinary defensive
code rather than as a requirement on five other files.

The failure is quiet in a specific way worth naming. An unreachable `elif` is not
a crash and not a wrong value; it is an omission, and the field it omits is one
whose absence the next turn accepts. `thought_signature=None` is a legal request.
Every layer behaves reasonably given its input, and the only place the loss is
visible is a comparison nobody performs: what went in against what came back out.

The other half of the lesson is that a round-trip whose two ends sit in the same
file reads as verified by inspection. You can see the encode at line 401 and the
decode a few hundred lines down, the symmetry is right there, and the asymmetry
lives in a shared file that neither end mentions. When checking a round-trip, the
question is never whether both ends agree with each other. It is whether anything
between them is preserving what they agree about.

## The fix that also works, and is worse

The one-word change in the shared aggregator (`elif` to `if`) also makes the
signature survive, and it keeps the existing tests green. It was measured rather
than assumed: `handle_content_block_delta` returns a single `typed_event`, so the
signature branch overwrites it and the `ReasoningTextStreamEvent` is lost.
Reasoning text would stop reaching streaming consumers.

Measured on `8a292d9`, same input (a thought part with `text="test reason"`,
`thought_signature=b"abc"`):

| | aggregated signature | `ReasoningTextStreamEvent` |
|---|---|---|
| `main` | dropped | emitted |
| `elif` to `if` in `streaming.py` | preserved | **lost** |
| adapter emits its own delta (the PR) | preserved | emitted |

Two candidate fixes both make the reported symptom disappear. One of them
silently breaks a second consumer, and no test distinguishes them. That is the
generalizable part: when more than one change makes the symptom go away, the
choice between them is a measurement, not a matter of taste, and the cheaper
change is the one more likely to be measured least.

## Trigger

Any streamed Gemini turn that produces a signed thought part. No configuration,
no scale, no concurrency. The user-visible effect is that thought signatures are
never returned to Gemini, which defeats the round-trip the adapter already
implements on both ends.

## Repro

Clean `python:3.12-slim` container against `8a292d9`, no API keys and no live
calls, with `importlib.util.find_spec(...).origin` checked on both sides to
confirm which copy of the package ran.

`test_stream_response_reasoning_signature_survives_aggregation` drives the real
`model.stream()` through the real `process_stream` and asserts three things: the
aggregated `reasoningText` carries the exact signature, the
`ReasoningTextStreamEvent` is still emitted (this is the assertion that fails on
the `elif`-to-`if` alternative), and feeding the aggregated message back through
`_format_request_content_part` reproduces the original signature bytes. Reverting
only the `gemini.py` hunks and keeping the test fails on unfixed source.

The two existing tests, `test_stream_response_reasoning` and
`test_stream_response_reasoning_and_text`, asserted the combined-delta chunk
shape. They passed while the aggregated message was wrong, which is how this
survived them: they pinned what the adapter emitted, and the defect is in what
the aggregator did with it.

**Not verified:** no live Gemini API calls were made, so all of the above is about
this SDK's own round-trip and not about how Google's service responds to a
request with a missing signature. The unit suite ran on Python 3.10 only. One
open question was flagged upstream rather than guessed at: if Gemini can attach a
complete signature to more than one part inside a single reasoning block,
`state["signature"] += ...` concatenates them, and feeding two signed parts
through the branch produces `'czE=czI='`, two base64 values joined rather than one
signature. Whether Gemini emits that shape could not be checked without live
calls. On `main` the same input yields no signature at all, so it is not a case
that works today.

**Corrected 2026-08-02.** As first written this entry localised the whole defect
to the shared aggregator and said both adapter ends were correct. That was wrong.
A second, independent drop sat in `stream()` on the same base commit, and a
reviewer found it four days after this was published. The miss is this entry's own
lesson turned on itself: we checked what the middle did with the delta the adapter
emitted, and never asked which parts the adapter declined to emit a delta for.
