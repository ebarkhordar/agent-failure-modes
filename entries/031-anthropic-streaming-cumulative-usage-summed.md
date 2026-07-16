# Summing a counter that is already cumulative counts everything twice

- **Repo:** microsoft/agent-framework
- **Surface:** `python/packages/anthropic/agent_framework_anthropic/_chat_client.py` (streaming usage), aggregating into `python/packages/core/agent_framework/_types.py::add_usage_details`
- **Class:** streaming & usage accounting
- **Report:** [issue #7143](https://github.com/microsoft/agent-framework/issues/7143) (deterministic repro, fix offered)

## Root cause

The Anthropic adapter emits a usage Content when it sees `message_start`, seeding the
counts the API reports there, and emits another when it sees `message_delta`.
`ChatResponse.from_updates` aggregates the stream through `add_usage_details`, which sums
all numeric values across the updates it receives.

Anthropic's `message_delta` usage is cumulative, not incremental: it restates the totals
for the message so far. Summing it onto the `message_start` seed therefore adds the same
tokens twice. The non-streaming path reads `message.usage` once and is correct, so the
defect is streaming-only. The sibling Gemini adapter in the same repo does not have it,
because it attaches usage to the final chunk only.

## Invariant violated

After a streaming call, `response.usage_details` equals the cumulative totals the API
reported, and matches what the non-streaming path yields for the same generation.

The general rule: a stream carries per-event values under one of two conventions,
incremental or cumulative, and an aggregator implements exactly one of them. Summing is
right for increments and double-counts cumulative values. Last-write-wins is right for
cumulative values and discards all but the final event's increments. Neither convention is
visible in the type. Under both, the event carries a dict of integers named `usage`, so
the aggregator type-checks and runs cleanly whichever convention the provider actually
chose.

That makes the aggregator's choice a claim about the provider's wire contract, settled
only by the provider's documentation, and it has to be re-established for each adapter
rather than inherited. A shared aggregation helper applied across providers is where this
goes wrong: `add_usage_details` is correct for adapters whose providers emit increments,
and reusing it silently asserts that this provider does too. The bug is not in the helper.
It is in the assumption that one aggregation convention can be shared across providers who
never agreed on one.

The second half is why it survived. Exact-value usage assertions existed only for the
non-streaming path and for the usage parser tested in isolation. The one test that touched
streaming usage asserted that `usage_details is not None`. An existence assertion on a
number tests plumbing, not arithmetic: it passes for every wrong value there is, and it
reports a covered line while covering nothing about what the line computes. The tests were
present, and the streaming totals had simply never been compared against a known answer.

## Trigger

Every streaming Anthropic call. `message_start` and `message_delta` are emitted on every
streaming request, so no input avoids it and no configuration disables it.

## Repro

Clean Docker `python:3.12-slim`, `--network none`, against HEAD `85c00fc`, with `__file__`
provenance printed
(`/usr/local/lib/python3.12/site-packages/agent_framework_anthropic/_chat_client.py`).
Real Anthropic SDK event objects (`BetaRawMessageStartEvent`, `BetaRawMessageDeltaEvent`,
`BetaMessageDeltaUsage`) fed through `_process_stream_event` into
`ChatResponse.from_updates`. No live provider calls. The anthropic SDK resolved to 0.116.0,
inside the package's own pin (`anthropic>=0.80.0,<0.117.0`).

With a delta carrying only `output_tokens`: `output_token_count` reports 26 against a true
25. With a delta that also carries prompt-side fields: input 200 against 100, cache-read
100 against 50, cache-creation 20 against 10. Baseline: the `packages/anthropic` suite is
green on pristine HEAD in the same image (148 passed), so the finding is additive rather
than environmental.

The cumulative semantics are Anthropic's published contract, not our inference: the
streaming documentation states that the token counts in the `message_delta` usage field
are cumulative.

**Not verified:** the second shape, a delta that also carries prompt-side fields, rests on
Anthropic's published streaming example rather than on our own observation of the live
wire. The unconditional case is the output-only one. Not bisected, so not claimed as a
regression, and no user-facing billing discrepancy was quantified.
