# Rebinding a loop variable edits the name, not the list

- **Repo:** strands-agents/harness-sdk
- **Surface:** `strands-py/src/strands/models/sagemaker.py::SageMakerAIModel.format_request` (the `tool_results_as_user_messages` payload option)
- **Class:** message-conversion boundaries
- **Report:** [issue #3299](https://github.com/strands-agents/harness-sdk/issues/3299) (deterministic repro, fix offered)

## Root cause

`format_request` walks the outgoing messages with `for message in payload["messages"]:`
and, for a tool result, assigns a newly built `{"role": "user", ...}` dict to `message`.
That assignment rebinds the local name to the new dict. The list still holds the object it
always held, so the tool-to-user conversion is a complete no-op and the formatted Body
still carries `role="tool"`.

The sibling branch a few lines up survives the same authorship because it happens to call
`message.pop()`, which mutates in place the dict the list already points at. So one branch
edits the payload and the neighbouring branch does not, and the difference is invisible in
the shape of the code: both read as "fix up this message".

The option has never worked. `git log --all -S` on the option name, followed across the
monorepo rename, shows exactly one commit ever touched the string, the one that introduced
SageMaker support in July 2025.

## Invariant violated

When `payload_config["tool_results_as_user_messages"]` is true, no message with
`role="tool"` appears in the JSON Body sent to the endpoint; each tool result is rewritten
as a user message. That is the option's entire documented purpose.

The general rule is that a `for` target is a name bound to an element, not a slot in the
container. Assigning to it rebinds the name and leaves the container exactly as it was.
Only mutating the object the name points at, or writing back through the container
(`lst[i] = ...`, which is what `enumerate` is for), changes anything the caller will see.
Mutation and rebinding are one character apart in appearance and opposite in effect, and
the language reports neither: no error, no warning, just a loop that runs to completion
and accomplishes nothing.

The part worth carrying to other codebases is what disguised it. An in-place branch
sitting next to a rebinding branch does not merely fail to reveal the bug; it actively
argues against it, because the reviewer confirms that the surrounding code really does
edit messages and generalizes that confirmation to the branch beside it. When auditing a
mutation loop, the question is not "does this branch look like it edits the element" but
"name the write that the caller can observe", asked separately of each branch.

The second half is a warning about ceremony. This option was declared in the payload
schema, documented in the class docstring, given a default, and deliberately excluded from
payload passthrough. Every surface that makes an option look supported was present and
correct for a year. All of them are cheap. The one that makes an option work is a single
write-back, and it is the only one that no amount of reading the declaration will verify.

Entry [032](032-hindsight-cooccurrence-swap-rebinds-outer-iterate.md) is the same mistake
with the opposite symptom: there, writing to the loop variable does not fail to reach the
container, it reaches the loop's own state and corrupts the iterations still to come.

## Trigger

Setting `tool_results_as_user_messages=True` against an endpoint that rejects
`role="tool"`, such as a TGI or vLLM deployment. That is exactly the failure the option
exists to prevent, so the trigger is a user following the documentation correctly.

## Repro

Clean Docker `python:3.12-slim`, `--network none`, against HEAD `4330a231`, with `__file__`
provenance printed
(`/usr/local/lib/python3.12/site-packages/strands/models/sagemaker.py`). No live SageMaker
endpoint is needed: `format_request` builds the wire Body offline.

Unpatched, with the flag set true, `role="tool"` is still present on the wire. As a
control, the same HEAD plus a one-line `enumerate`-and-write-back change: the tool message
becomes `role="user"` with content `Tool call ID 't1' returned: 72F`. Root cause and fix
are both confirmed by execution rather than by reading.
