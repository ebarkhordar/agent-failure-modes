# A sanitiser retired because a new guard subsumes it, where the guard fires on the branch the leak does not arrive on

- **Repo:** vectorize-io/hindsight
- **Surface:** `hindsight-api-slim/hindsight_api/engine/reflect/agent.py`: the
  `if not saw_tool_call` guard at `:919` (raise at `:924`, `saw_tool_call = True` at
  `:988`) and the unvalidated `answer = args.get("answer", "").strip()` at `:1268`,
  against the sanitisers deleted by
  [PR #3013](https://github.com/vectorize-io/hindsight/pull/3013) (merged 2026-07-28)
- **Class:** message-conversion boundaries
- **Report:** triaged publicly on
  [issue #3048](https://github.com/vectorize-io/hindsight/issues/3048#issuecomment-5124842980)
  (open, filed by another contributor whose production traces are the observation). No PR
  from us: the remedy is a choice between validating the `answer` value and removing the
  id-parameter boundary from the `done` tool, which is the maintainers' design call, and
  the reporter raised the second option explicitly as an option rather than a
  recommendation.

## Root cause

Reflect is driven by a `done` tool call whose arguments carry the user-facing `answer`
alongside sibling id fields (`memory_ids` and friends). Before #3013, a set of helpers
(`_clean_done_answer`, `_unwrap_leaked_done_arguments`,
`_strip_trailing_id_json_object`, `_clean_answer_text`) scrubbed tool-call syntax out of
the answer string on the way through. #3013 replaced that with a loud failure and
deleted all of them, on a stated premise:

```python
# ``done`` is a structured tool call: trust its ``answer`` field verbatim.
# Sibling id fields (memory_ids, ...) live in their own arguments and are
# validated separately below -- they can't bleed into a parsed answer string.
answer = args.get("answer", "").strip()
```

The guard that replaced the scrubbing is at `:919`, under `if not saw_tool_call`. It
covers the case #3013 was written for, a transport that strips tool definitions so the
model answers in free text that imitates a `done` payload.

The corruption the reporter observes arrives on the other branch. The provider returns a
well-formed `done` tool call, with a correctly populated `memory_ids` argument, and the
`answer` string value ends in a fragment of a second, XML-flavoured tool-call
serialisation (`</answer>` followed by `<parameter name="memory_ids">["<uuid>", ...`).
`saw_tool_call` is `True`, so nothing raises, `:1268` takes the value verbatim, and it is
persisted into mental model content and re-parsed into `structured_content`, so one
occurrence corrupts both surfaces. Nothing downstream looks: the only other
`args.get("answer")` in the module is `:1547`, inside `_summarize_input`, which builds a
log preview.

The part that inverts the reporter's diagnosis is that restoring the deleted sanitisers
would not have helped. Measured against the observed shape, the pre-#3013 helpers return
it byte for byte:

```
=== 3048 observed shape
    stripped? NO
    output is byte-identical to input (184 chars)
=== control: trailing ids colon    -> YES  'The team migrated the billing service.'
=== control: trailing json object  -> YES  'The team migrated the billing service.'
=== control: fenced json           -> YES  'The team migrated the billing service.'
```

The patterns say why. `_TRAILING_IDS_PATTERN` requires `memory_ids` followed by `=` or
`:`, `_LEAKED_JSON_SUFFIX` requires a fenced block, and
`_strip_trailing_id_json_object` requires the text to end in `}`. All three were built
for a JSON-flavoured leak. The observed one is XML-flavoured and matches none of them.
So this is a gap that predates #3013 rather than a regression from it, and reverting
#3013 would leave the two already-corrupted mental models exactly as they are.

## Invariant violated

**Retiring a defensive layer because a new guard subsumes it is a claim about two
coverage sets, and it is settled by comparing them, not by comparing intentions.** The
deleted sanitisers ran on every answer string. The guard runs only when no tool call was
parsed. The set difference is the entire tool-calling path, which is the path every
working provider takes, so the replacement narrowed coverage to the failure mode that
motivated it while the change reads as strictly stronger: it turns silent salvage into a
hard error. A guard and a sanitiser are not interchangeable because they are not indexed
by the same question. One asks whether the transport worked; the other asks what the
bytes contain.

**A structured field is not a validated field.** "It came out of a parsed tool call"
records where the bytes were found, not what is in them. It is a claim about the
producer's serialiser, and the parser that consumes the call cannot check it: a model
free-running two tool-call formats at once can put the boundary between `answer` and its
siblings anywhere, because in the token stream that boundary is a convention rather than
a frame. The schema separates the parameters; the generation does not have to.

What makes this one cheap to adjudicate, and worth recording as a habit, is that **the
falsified assumption was written down twice**, in the comment at `:1265-1267` and in
#3013's own commit message ("A parsed tool call can't bleed its sibling id fields into
the answer string"). A comment that states an invariant is the highest-value place to
point a counterexample, for two reasons. It records what the author believed rather than
what the code does, so it is the one artifact a reader can disagree with directly. And
it propagates: the next person to touch `_process_done_tool` inherits the belief in the
file, tests against it, and has no reason to re-derive it. An assumption in a comment is
load-bearing exactly as long as nobody has produced the counterexample, and the trace
here is one.

## Trigger

A provider that blends two tool-call serialisations inside one generation. In the
reporter's self-hosted deployment (`anthropic/claude-haiku-4-5` through a LiteLLM proxy,
not the Vertex/gpt-oss path #3013 addressed): 2 occurrences in about 172 generations on
2026-07-22 and 3 in about 830 on 2026-07-27, each persisting into one mental model, the
two worst carrying 133 and 49 leaked ids. Low rate and silent, because the refresh
reports success.

## Repro

What was executed: the pre-#3013 sanitisers, extracted from `6fe0dd6` (the parent of
`678ca0e9`) and run in a clean `python:3.12-slim` container with `--network none` and a
read-only scratch mount, stdlib only, against the leak shape from the reporter's trace
plus the three shapes those helpers were written for. Output above.

The structural claims were confirmed at HEAD `cc1eaee`: the guard's placement and line
numbers, the unvalidated read at `:1268`, and the absence of any downstream inspection.
The last of those is an enumeration, so it was answered by parsing the module rather than
by counting string matches.

**Not verified, and this bounds the entry:** the generation itself was not reproduced.
The leak shape, the rates and the two corrupted mental models are the reporter's
observations, not ours, so nothing here is a claim about how often this happens or about
which layer blends the two serialisations. The write past `ReflectAgentResult` into
`mental_models.content` was not traced either.

Verified 2026-07-30 at HEAD `cc1eaee`. Reported, not fixed upstream at the time of
writing.
