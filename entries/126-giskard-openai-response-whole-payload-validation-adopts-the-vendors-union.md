# Validating the whole vendor payload adopts the vendor's union as your own contract, and this one declares two of the SDK's twenty-eight output item types

- **Repo:** Giskard-AI/giskard-oss
- **Surface:** `libs/giskard-llm/src/giskard/llm/translators/openai_response.py:87`
  (`OpenAIResponseTranslator.from_openai`), against `ResponseOutputItem` at
  `libs/giskard-llm/src/giskard/llm/types/response_result.py:11` and the sibling
  `from_google` in `translators/google_response.py`
- **Class:** message-conversion boundaries
- **Status:** unreported upstream. No issue, no PR. This entry records the reproduction.

## Root cause

`from_openai` is a single statement:

```python
@staticmethod
def from_openai(raw: "Response") -> ResponseResult:
    return ResponseResult.model_validate(raw.model_dump())
```

The destination field is typed:

```python
ResponseOutputItem = ResponseOutputMessage | ResponseFunctionToolCall

class ResponseResult(_BaseModel):
    outputs: list[ResponseOutputItem] = Field(
        validation_alias=AliasChoices("output", "outputs")
    )
```

Two members. The OpenAI SDK's own `ResponseOutputItem` union has twenty-eight, among them
`reasoning`, `web_search_call`, `file_search_call`, `mcp_call`, `code_interpreter_call` and
`image_generation_call`. Because the translator validates the entire dumped payload rather
than reading the parts it understands, every item in `output` has to satisfy the
two-member union, so twenty-six of the twenty-eight abort the translation with a pydantic
`ValidationError` before any of the response reaches the caller.

The one that matters in practice is `reasoning`. Any reasoning model on the Responses API
emits a `reasoning` item ahead of its message, so the translator fails on the ordinary
output of the models it exists to support, not on an exotic tool configuration.

The failure is also total rather than partial. The offending item is item zero of a list
whose remaining entries are perfectly well formed messages, and the assistant's actual text
is present in the payload the whole time. Nothing selects it, because validation runs over
the container.

The sibling translator in the same package, producing the same `ResponseResult`, does the
opposite. `from_google` walks `raw.steps` and appends only for `item.type == "model_output"`
and `item.type == "function_call"`, so an unrecognised step is skipped and the recognised
content still arrives. Two translators, one target type, opposite answers to the same
question about an unknown item.

## Invariant violated

**Handing a whole vendor payload to `model_validate` converts your type annotation from a
description of what you handle into an enforcement of what the provider is allowed to
send.** Read at the declaration site, `list[ResponseOutputMessage | ResponseFunctionToolCall]`
looks like documentation: these are the two output kinds this library does something with.
Pydantic makes it a gate on the wire format instead. The library's own scope and the remote
API's surface are different sets, and whole-object validation is the operation that quietly
equates them.

**A translator's contract is to render what it can, so an item it does not recognise is
missing information and not an error.** Rejecting the response is strictly worse than
returning the message and ignoring the reasoning item, because the caller wanted the
message and there was never a version of this call where a `reasoning` item was going to be
useful to it. Selective construction and whole-payload validation differ exactly here: one
degrades and the other refuses, and the choice is invisible in a one-line function.

**A union over another party's taxonomy is complete only on the date it was written, and it
goes stale without anyone editing it.** Nothing in this repository changed to break this.
The two-member union has been two members since `c3399ae2a` (2026-04-29), the commit that
introduced the library, and `820893a6e` (#2437) later swapped one member type for another
without touching the arity. The vendor added item kinds on their own schedule. This is the
failure mode with no diff to review and no test to fail: the code that becomes wrong is
code nobody touched.

**Sibling implementations of one conversion are the cheapest available oracle, and
disagreement between them is a finding on its own.** `from_google` and `from_openai` return
the same type from the same package for the same purpose. Reading them side by side names
the defect without reference to any specification, and the one whose behaviour is
defensible is the longer one.

## Trigger

Any call through `OpenAIResponseTranslator.from_openai` whose response contains an output
item outside `{message, function_call}`. Reasoning models reach it on ordinary completions;
built-in tools (web search, file search, code interpreter, MCP, image generation) reach it
whenever they are enabled. No configuration of this library avoids it, because the
translator has no branch that could.

## Invariant discharge

The member counts are enumerated, not searched: `typing.get_args` over the SDK's
`ResponseOutputItem` (unwrapping the `Annotated` wrapper) yields 28 members, and over
giskard's yields 2. The claim that no earlier attempt or deliberate exclusion exists is
from `git log -G` with the probe strings escaped for regex metacharacters, on an unshallowed
clone (`git rev-parse --is-shallow-repository` returns false), one pathspec per invocation
because `--follow` accepts only one. The first probes returned zero hits against a
`grep -cF` of 1 in the working tree, which is a broken probe rather than an absent history;
run correctly they return the two commits adjudicated above, and reading both diffs shows a
member rename and no change in arity.

## Repro

Clean `python:3.12-slim` container at `b61e72a5b`, `openai==2.46.0`, `pydantic==2.13.4`,
giskard-core and giskard-llm installed from the checkout so that `__file__` resolves under
`site-packages` rather than the source tree. No API key and no network (`--network none`):
the `Response` is built locally with `model_construct`, which is what the SDK would hand
back after decoding a reasoning model's reply.

The differential is the same call twice, changing only the output list:

```
A  output=[message]              -> OK, outputs=1, output_text='hi'
B  output=[reasoning, message]   -> ValidationError: 7 validation errors for ResponseResult
                                    output.0.ResponseOutputMessage.type
                                    Input should be 'message'
                                    [type=literal_error, input_value='reasoning']
```

**Not verified:** no call was made to a real OpenAI endpoint, so the reasoning item is
modelled from the SDK's own type rather than captured from the wire. Only `reasoning` was
executed; the other twenty-five excluded kinds are read off the two unions rather than run.
Whether the maintainers consider the strictness deliberate is unknown, since it is
unreported and nothing in the repository states an intent either way.

Verified 2026-08-04 at `b61e72a5b`.
