# Two index sets over one list, one reversed against its own deletions and one not

- **Repo:** UKGovernmentBEIS/inspect_ai
- **Surface:** `inspect_ai/model/_compaction/edit.py` (`CompactionEdit` apply, `_apply_server_tool_clearing`)
- **Class:** indexing, ordering & counting contracts
- **Fix:** [PR #4529](https://github.com/UKGovernmentBEIS/inspect_ai/pull/4529)
  (in review; issue [#4528](https://github.com/UKGovernmentBEIS/inspect_ai/issues/4528))

## Root cause

`CompactionEdit(keep_tool_inputs=False)` clears tool inputs from a message list.
It first enumerates `result` once to collect every tool use it will clear, then
splits them into two work lists: `client_to_clear` (tool uses stored as whole
messages that get deleted) and `server_to_clear` (server-side tool uses embedded
in an assistant message's content list, edited in place). Both lists hold
positional indices captured in that single pre-mutation enumeration.

The client-side loop knows deletion invalidates indices and defends itself: it
iterates `reversed(client_to_clear)` with the comment "process in reverse to
preserve indices", so each `del result[i]` only shifts entries after `i`, which
have already been handled. But `server_to_clear`'s message index comes from the
same pre-deletion enumeration and is consumed after those deletions run. Nothing
rebases it. Reversal protects the client list from its own mutations; it does
nothing for a second index set that outlives the same mutations.

So one `del result[1]` moves the server-side assistant message from index 2 to
index 1 while `server_to_clear` still records 2. Index 2 now points at the
trailing user message, whose content is a plain string, so its content list is
empty, and `content_list[content_idx] = cleared_tool_use` raises
`IndexError: list assignment index out of range`.

## Invariant violated

Every positional index into a list must be consumed before any length-changing
mutation of that list, or explicitly rebased after it. A single enumeration that
feeds two consumers cannot be made safe by hardening one consumer: reversing the
client deletions preserves the client indices and leaves the server indices
pointing wherever the deletions pushed them. Two independent index sets over one
mutating list need two independent invalidation strategies, or one deletion pass
that both share.

The `IndexError` is the fortunate outcome. Where the shifted target happens to be
another assistant message with list content, the stale index does not raise: it
writes a cleared server tool use into the wrong message, silently. The crash is
the lucky case; corruption is the unlucky one.

## Trigger

`keep_tool_inputs=False` together with at least one server-side tool use (an
assistant carrying a `ContentToolUse`, e.g. `web_search`) and a client-side tool
call whose message is deleted ahead of it. A documented option plus an ordinary
tool combination. The stock edit suite never pairs the two, so the path ships
untested.

## Repro

Clean `python:3.12-slim` container against `6dbab88`, `pip install -e '.[dev]'`,
provenance by sha256 identity between source tree and installed module. History:
one client-side `bash` call and its result, one assistant carrying a server-side
`web_search` `ContentToolUse`, a trailing user message.

```
keep_tool_inputs=True    -> 4 messages, clean (control)
keep_tool_inputs=False   -> IndexError at edit.py content-list assignment
```

Baseline on HEAD is green: the compaction edit test files pass and skip as
normal, because none combines `keep_tool_inputs=False` with a server-side tool
use.
