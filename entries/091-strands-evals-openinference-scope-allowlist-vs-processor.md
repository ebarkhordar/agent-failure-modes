# A format filter keyed on instrumentation scope drops the spans a processor converted, because a SpanProcessor cannot change scope

- **Repo:** strands-agents/evals
- **Surface:** `src/strands_evals/mappers/openinference_session_mapper.py`, the span
  filter at line 96, against the allowlist in
  `src/strands_evals/mappers/constants.py:11-19`
- **Class:** message-conversion boundaries
- **Report:** triaged publicly on
  [strands-agents/harness-sdk#3432](https://github.com/strands-agents/harness-sdk/issues/3432#issuecomment-5063589620)
  (open; fix options adjudicated in the comment). No PR from us: the fault code lives
  in `strands-agents/evals`, a repo this lane had not gated for contribution, so the
  reproduction was contributed as triage rather than as a patch.

## Root cause

`OpenInferenceSessionMapper` decides which spans it will convert by testing each
span's instrumentation scope name against a frozen allowlist:

```python
openinference_spans = [s for s in spans if self._get_scope_name(s) in SCOPES_OPENINFERENCE_FAMILY]
```

`SCOPES_OPENINFERENCE_FAMILY` holds exactly two names,
`openinference.instrumentation.langchain` and
`openinference.instrumentation.smolagents`.

What makes a Strands span OpenInference-shaped, however, is
`StrandsAgentsToOpenInferenceProcessor`, which is a `SpanProcessor`, not a tracer. A
`SpanProcessor` runs in `on_end` and rewrites attributes; it has no ability to change
a span's instrumentation scope, which is fixed at creation by whichever tracer made
the span. For Strands that tracer is `strands.telemetry.tracer`. So every span the
processor converts arrives at the mapper carrying a scope that is not in the
allowlist, is filtered out before any conversion runs, and the mapper returns
`Session(traces=[])`.

The same file already knows the name it is missing: `constants.py:11` defines
`SCOPE_STRANDS = "strands.telemetry.tracer"`, and the family set two definitions
below simply omits it.

The repair proposed in the issue, testing
`scope_name.startswith("openinference.instrumentation.")`, does not fix this case,
because the real producer's scope does not carry that prefix. It widens the allowlist
along the axis the bug is not on.

## Invariant violated

A filter that asks "is this payload in the format I convert" must key on a marker
controlled by whatever produces that format. An instrumentation scope is stamped by
the tracer, so it identifies the *producer*; when the format is applied afterwards by
a processor layered downstream of that producer, scope answers a different question
than the one the filter is asking. The correct key is the capability marker the
converting layer actually writes, here the `openinference.span.kind` attribute the
processor stamps on every span it converts.

Generally: whenever a transform is applied by a decorator or processor stage
downstream of the thing that creates the data, any consumer that gates on producer
identity will silently drop exactly the payloads the transform exists to serve. The
failure is invisible in both directions. An allowlist miss yields an empty result
rather than an error, so the consumer reports success on zero data, and the producer
side looks correct because the spans really are well formed. An allowlist keyed on
identity also cannot be repaired by extending it, only by re-keying it: adding names
fixes the instances someone reports and leaves the next integration to rediscover the
bug.

## Trigger

Any OpenTelemetry pipeline where Strands agent spans are converted by
`openinference-instrumentation-strands-agents` and then mapped through
`OpenInferenceSessionMapper`. The spans are valid OpenInference and carry
`openinference.span.kind`; the session comes back with no traces and no error.

## Repro

Clean `python:3.12-slim` container, `strands-agents/evals` installed from HEAD
`771a3fe` (`strands_evals` 1.0.3). A span carrying scope `strands.telemetry.tracer`
maps to `traces=0`, while a langchain-scoped span maps to `traces=1`. On that same
real span, `startswith("openinference.instrumentation.")` is `False` and the
`openinference.span.kind` attribute is present, which is the argument in two lines:
the prefix test misses it, the attribute test catches it. End to end, a span emitted
by the real Strands tracer, routed through the real
`StrandsAgentsToOpenInferenceProcessor` and an in-memory exporter, keeps scope
`strands.telemetry.tracer` and carries `openinference.span.kind`, confirming that the
processor converts the attributes and leaves the scope alone.

Verified 2026-07-23 at HEAD `771a3fe`. Re-checked 2026-07-26: `771a3fe` is still the
tip, and `SCOPES_OPENINFERENCE_FAMILY` still omits `SCOPE_STRANDS`, so the defect is
live. Not fixed upstream at the time of writing.
