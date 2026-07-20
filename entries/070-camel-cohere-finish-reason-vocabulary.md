# A provider's finish reason passed through raw, into a field whose contract enumerates a disjoint set of values

- **Repo:** camel-ai/camel
- **Surface:** `camel/models/cohere_model.py::CohereModel._to_openai_response`
- **Class:** message-conversion boundaries
- **Fix:** [PR #4202](https://github.com/camel-ai/camel/pull/4202) (in review;
  issue [#4201](https://github.com/camel-ai/camel/issues/4201))

## Root cause

`_to_openai_response` normalizes a Cohere v2 response into an OpenAI-shaped
`ChatCompletion` and assigns the finish reason directly:

```python
finish_reason=response.finish_reason
```

The two vocabularies do not overlap at any point. Cohere's `V2ChatResponse`
declares `Literal['COMPLETE', 'STOP_SEQUENCE', 'MAX_TOKENS', 'TOOL_CALL',
'ERROR', 'TIMEOUT']`; OpenAI's `Choice.finish_reason` declares
`Literal['stop', 'length', 'tool_calls', 'content_filter', 'function_call']`.
The intersection is empty, so every value this adapter can emit is off-contract,
not merely some edge case among them.

Nothing catches it, because the object is built with
`ChatCompletion.construct()`, which skips pydantic validation by design. The
illegal value is written into a field that is typed to forbid it, and the
program continues with no error and no log line. Sibling adapters in the same
codebase already do the translation: `XAIModel._map_finish_reason` and
`AWSBedrockConverseModel._map_stop_reason` both map into the OpenAI vocabulary,
so the Cohere adapter is the outlier rather than the precedent.

## Invariant violated

A field's contract is its value set, not its name and type. An adapter that
copies a same-named, same-typed field across a format boundary has translated
nothing: `str` to `str` type-checks perfectly while carrying a value the target
format has no meaning for. The type system cannot help here, because `Literal`
constraints are the whole specification and the one construction path used
(`construct()`) is defined to bypass them. Where several provider adapters
normalize into one shared response shape, each must independently satisfy that
shape's semantics, since consumers read the shape and never the provider behind
it.

The generalization worth keeping: when you see an assignment whose right side
and left side are named the same thing across an interoperability boundary,
that is the most likely place for an untranslated value to live, precisely
because the name match makes the code look already correct.

## Trigger

Every Cohere completion, without exception. There is no input that produces a
compliant value, which is why no test caught it: a test asserting "the finish
reason is passed through" passes, and only a test asserting membership in
OpenAI's value set fails.

## Repro

Clean `python:3.11-slim` container, camel installed from source at the branch,
`cohere` 5.21.1 (camel pins `cohere>=5.11.0,<6`), with `__file__` provenance
printed on both sides. Each of the six values Cohere can return was driven
through the real `_to_openai_response`: 6 of 6 emitted a value outside OpenAI's
legal set before the fix, 0 of 6 after. The repo's own suite, run the way CI
runs it (`pytest test/models/test_cohere_model.py --fast-test-mode -m "not
heavy_dependency"`), gives 7 failed / 4 passed / 5 skipped on master with the
new tests applied, and 11 passed / 5 skipped on the branch.

Scope was settled by parsing the module rather than grepping it:
`_to_openai_response` is the only writer of `finish_reason` in
`cohere_model.py`, both `_run` and `_arun` route through it, and
`CohereModel.stream` returns `False`, so there is no second conversion path.

**Stated plainly, and stated in the PR too:** this is a contract-compliance
defect, not a crash. Parsing the whole package for code that consumes a
`finish_reason` *value* (comparisons, subscript keys, `.get()` keys) returns
exactly two sites, both in `xai_model.py` and both about xai's own responses, so
no camel code path changes behavior today. The value reaches users through
`response.info['finish_reasons']`, and camel's own config docstrings
(`minimax_config.py:50`, `samba_config.py:98`, and 14 others) document the
OpenAI vocabulary as the contract, describing a `finish_reason="length"` the
Cohere backend could never produce. No live Cohere API call was made; the value
set comes from executing the pinned SDK's own type, and the conclusion that
camel therefore emits a non-compliant value follows from the pass-through
statically rather than from a captured wire response.
