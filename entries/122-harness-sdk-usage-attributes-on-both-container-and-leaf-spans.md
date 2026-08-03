# The same token usage is written to a span and to its own children, so any backend that sums the trace doubles the bill

- **Repo:** strands-agents/harness-sdk
- **Surface:** `strands-py/src/strands/telemetry/tracer.py:804-810` (`Tracer.end_agent_span`, the
  `invoke_agent` container span) together with `:438-446`
  (`Tracer.end_model_invoke_span`, the `chat` leaf span)
- **Class:** streaming & usage accounting
- **Report:** measured publicly on
  [issue #3602](https://github.com/strands-agents/harness-sdk/issues/3602#issuecomment-5161084586)
  (open, filed by another user). No PR: the shape of a fix here is a maintainer decision that
  has already been made once, the other way, and a large open PR is rewriting the same block.

## Root cause

An agent run emits a span tree. `invoke_agent` is the container; under it sit one
`execute_event_loop_cycle` per turn, and under those the `chat` spans that are the actual
model calls. Both ends of that tree write `gen_ai.usage.*`: the `chat` leaf writes the usage
of its own call, and the container writes `event_loop_metrics.accumulated_usage`, which is by
construction the sum of the `chat` spans beneath it. The same tokens are therefore present
twice in one trace, once as leaves and once as their own total.

Measured on `3cddf74`, offline, with a mocked model provider and an in-memory span exporter,
two model calls of 100/10 and 200/20 tokens with a tool call between them:

```
- invoke_agent Strands Agents   input=300 output=30 total=330
  - execute_event_loop_cycle    (no usage attrs)
    - chat                      input=100 output=10 total=110
    - execute_tool add          (no usage attrs)
  - execute_event_loop_cycle    (no usage attrs)
    - chat                      input=200 output=20 total=220

real model usage      input=300 output=30
naive trace-tree sum  input=600 output=60
```

Two details narrow it. First, the surface is one function, not two: parsing the package with
`ast` for writes of `gen_ai.usage.*` returns exactly two sites, `end_model_invoke_span` and
`end_agent_span`. `end_event_loop_cycle_span` writes none, and neither do the multiagent and
swarm spans, so the intermediate layer of the tree is already clean.

Second, this ground is settled rather than virgin. Issue #2010 reported the same inflation and
[PR #2017](https://github.com/strands-agents/harness-sdk/pull/2017) resolved it by adding an
opt-in, `OTEL_SEMCONV_STABILITY_OPT_IN=gen_ai_use_latest_invocation_tokens`, which swaps
`accumulated_usage` for the latest invocation's usage rather than removing the attributes. The
opt-in does not close the double count. One agent reused across two turns, 100/10 then 200/20:

| | call 1, `invoke_agent` / `chat` | call 2, `invoke_agent` / `chat` |
|---|---|---|
| default | 100 / 100 | 300 / 200 |
| opt-in set | 100 / 100 | 200 / 200 |

With the opt-in the container mirrors exactly one leaf, so the inflation is pinned at 2x.
Without it the container carries the running total, so a long session drifts past 2x. The
opt-in changes the growth rate of the error, not its existence.

## Invariant violated

**A quantity that is summed across a tree must be written at exactly one depth.** Token usage
is additive, and every observability backend that shows a trace total adds up what it was
given. Writing the same tokens on a parent and on its children does not give a consumer two
views to choose between, it gives them one number that is wrong unless they happen to know
which depth to ignore. The choice of depth is a real design question, leaves or container, but
it is a choice of one.

**The trap is that both writers are individually correct.** The leaf reports what its own call
cost. The container reports what the run cost. Read one span at a time, each is exactly the
number a reader wants, which is why this survives review and why the code carries no sign of a
mistake. The defect exists only in the relation between them, and a relation is not visible at
either site. Nothing in a per-span assertion can fail.

**An opt-in that rescopes a duplicated value does not undo the duplication.** The earlier fix
narrowed the container's number from "all turns" to "the last turn", which reads as a fix for
inflation and measures as one on a single-turn run, the shape most tests take. It is worth
separating the two questions a duplicate raises: which value should this field hold, and should
this field exist here at all. Answering the first can close a report while leaving the second
untouched, and the residue is a smaller, steadier error that is harder to notice than the
original.

**Prior art on the exact lines is a verdict to answer, not a search result to log.** The
reported fix here, deleting the attributes, is the option the maintainers considered and
declined in #2017; it also takes data away from anyone reading tokens off `invoke_agent`
today. Knowing that changes the contribution from a patch into a question about intent.

## Trigger

Any traced run with at least one model call, in any OTLP backend that aggregates usage over a
trace. It does not need a tool call or multiple turns; a single call already writes its tokens
at both depths. It presents as a billing or cost-dashboard discrepancy rather than as a
telemetry bug, which is why the report arrived from operators comparing a dashboard against a
provider invoice.

## Repro

Clean `python:3.13-slim` container, `--network none`, `pip install -e ./strands-py` at
`3cddf74`, with `MockedModelProvider` and `InMemorySpanExporter` so no credentials or network
are involved. The script prints every finished span with its `gen_ai.usage.*` attributes and is
included in the public comment. The two-turn table is the same script with the tool dropped and
the agent invoked twice. The writer enumeration was done by parsing the package with `ast` for
string constants beginning `gen_ai.usage.`, not by grep, so it is a complete set of write sites
rather than a sample of matching lines.

**Not verified:** what Langfuse does downstream with `langfuse.observation.type = "span"`,
which `end_agent_span` sets on the container span when a Langfuse endpoint is configured. That
attribute was confirmed to land on `invoke_agent`, but no Langfuse account was available to
check whether it changes how that backend aggregates, so the report's claim that Langfuse is
affected is neither confirmed nor refuted here. Opik and Datadog have no equivalent branch. The
reporter's second claim, that `execute_event_loop_cycle` also carries usage attributes, is
wrong and was corrected in the comment.

Verified 2026-08-03 against `3cddf74`.
