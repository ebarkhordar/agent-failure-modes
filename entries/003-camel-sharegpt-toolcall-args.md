# Tool-call arguments corrupted in Hermes/ShareGPT round-trip

- **Repo:** camel-ai/camel
- **Surface:** Hermes ShareGPT message serialization round-trip
- **Class:** message-conversion boundary
- **Fix:** [PR #4170](https://github.com/camel-ai/camel/pull/4170)

## Root cause

Round-tripping a tool call through the ShareGPT/Hermes format silently
dropped or mangled arguments containing quotes or booleans — the
serialization and deserialization sides disagreed about escaping.

## Invariant violated

`deserialize(serialize(x)) == x` for every value the schema admits, not just
for the easy ones. A round-trip that only survives alphanumeric arguments is
a data-loss bug, not a format.

## Trigger

Any tool call whose arguments contain quote characters or JSON booleans —
i.e., ordinary real-world tool calls.

## Repro

Round-trip a tool call with `{"query": "say \"hi\"", "verbose": true}`;
compare input and output. Regression test in the PR.
