# A widened parameter that only one of two entry points carries, so a cache point survives on one path and is flattened away on the other

- **Repo:** strands-agents/harness-sdk
- **Surface:** `strands/agent/agent.py:943` (`Agent.structured_output_async`), against
  `strands/event_loop/streaming.py:523` and the `structured_output` / `stream` signatures of
  the 13 providers under `strands/models/`
- **Class:** message-conversion boundaries
- **Report:** reproduced publicly on
  [issue #3496](https://github.com/strands-agents/harness-sdk/issues/3496#issuecomment-5098513116)
  (open), adding a credential-free reproduction and correcting the scope in two directions. No
  PR from us yet: the repair spans `Agent`, the provider interface and
  `BedrockModel.structured_output`, and how far to take it is the maintainers' call.

## Root cause

A system prompt used to be a string. Prompt caching made it a list of typed blocks, because a
cache point is a marker positioned *between* blocks and there is nowhere to put it inside a
string. `Agent` therefore holds both: `system_prompt_content` is the block list, and
`system_prompt` is the flattened text.

`Agent.structured_output_async` passes only the flattened one. At `agent.py:943` it calls the
model's `structured_output` with `system_prompt=...` and nothing else, so every non-text block
is gone before any provider is reached:

```
agent.system_prompt_content : [{'text': '...'}, {'cachePoint': {'type': 'default'}}]
agent.system_prompt         : '...'
structured_output received  : system_prompt='...'   kwargs={}
cachePoint reached provider : False
```

Feeding both shapes to `BedrockModel.format_request` shows what that costs in the Converse
body:

```
system_prompt_content forwarded : [{"text": "..."}, {"cachePoint": {"type": "default"}}]
flattened string only           : [{"text": "..."}]
```

The block does not error. It is simply absent, so the request is well formed and uncached.

Two findings widen the picture, and the first inverts the report's severity claim. **The
recommended path is not affected.** `agent.structured_output()` is deprecated in favour of
passing `structured_output_model` on the invocation, and that path never reaches
`Model.structured_output` at all. It runs the ordinary event loop, which calls
`model.stream(..., system_prompt_content=system_prompt_content)` at
`event_loop/streaming.py:523`. Both paths in one process, same recording model, same prompt:

```
A. agent.structured_output(Output, ...)         entry point: structured_output   cachePoint survived: False
B. Agent(..., structured_output_model=Output)   entry point: stream              cachePoint survived: True
```

**The gap is not Bedrock-specific, and it is not one missing forward.** Parsing every provider
under `strands/models/`, `structured_output` declares `system_prompt_content` in 0 of 13
signatures while `stream` declares it in 3 (`model.py`, `bedrock.py`, `litellm.py`); the other
ten absorb it into `**kwargs` and ignore it. Separately, `BedrockModel.structured_output`
calls `self.stream(messages=..., system_prompt=system_prompt, tool_choice=..., **kwargs)`
without passing `system_prompt_content` on, so having `Agent` forward the block list would
still stop one hop short unless that call changes too.

## Invariant violated

When a parameter is widened from a scalar to a structured shape, the old scalar does not go
away. It becomes a lossy projection of the new one that still type-checks everywhere. Every
call site that keeps passing it stays green: the argument has the right type, the request is
well formed, and the only evidence of loss is a feature that quietly does not happen. So the
widening is not finished when the new parameter exists and one path carries it. It is finished
when every path that can reach the boundary either carries the new shape or is provably unable
to produce one.

The enumeration is the work, and it is mechanical. A capability parameter lives on an
interface with sibling methods, so the two facts to establish are which siblings *declare* it
and which forwarders *pass it on*. Both are answered by parsing signatures and call bodies,
never by reading the one file the report points at, and `**kwargs` is exactly what makes a
casual count misleading: a method that absorbs the parameter and drops it is indistinguishable
at the call site from one that honours it. Here the counts are 0 of 13 and 3 of 13, and the
one-hop-short forward inside the single provider that does declare it is the part a fix aimed
at the reported line would miss.

The corollary is for anyone reading a bug report rather than writing one. Check whether the
*deprecated* path is the broken one before accepting a claim that there is no working
replacement. A deprecation usually means a second path was built, and a second path built
later is the one more likely to have been given the newer parameter. Here it had been, which
turns the reported impact around: the advice stops being "wait for a fix" and becomes "move
off the deprecated call".

## Trigger

`agent.structured_output()` or `agent.structured_output_async()` on an `Agent` configured with
a `system_prompt_content` that contains any non-text block. The documented case is a
`cachePoint`, so the practical cost is that prompt caching is silently off for structured
output requested through that entry point, and the full system prompt is billed on every
call. Passing `structured_output_model` on the invocation instead is unaffected.

## Repro

`python:3.12-slim`, `SETUPTOOLS_SCM_PRETEND_VERSION=1.36.0`, `main` at `b84840ba`, with no
Bedrock account and no network. A `Model` subclass that records the arguments it is handed
replaces the provider, which is enough both to show the drop at `agent.py:943` and to run the
two entry points side by side in one process. The Converse-body consequence comes from calling
`BedrockModel.format_request` directly on each of the two shapes. The 0-of-13 and 3-of-13
signature counts are an AST parse of every module under `strands/models/`, not a text search.

**Scope of what was measured.** No live model was called and no Bedrock request was sent, so
the caching cost is read off the Converse body that `format_request` produces rather than
observed as a billing difference.
