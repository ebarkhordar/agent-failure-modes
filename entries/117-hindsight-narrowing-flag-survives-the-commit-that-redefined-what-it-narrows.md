# A narrowing flag survives the commit that redefined what it narrows, so an unretained suffix is cut down as though it were the whole transcript

- **Repo:** vectorize-io/hindsight
- **Surface:** the Claude Code retention plugin:
  `hindsight-integrations/claude-code/scripts/retain.py` (the `retain_full_window` computation,
  the `retainToolCalls` default at `:149`, the checkpoint advance at `:128-133`) and
  `hindsight-integrations/claude-code/scripts/lib/content.py`
  (`slice_last_turns_by_user_boundary`)
- **Class:** indexing, ordering & counting contracts
- **Report:** reproduced publicly on
  [issue #3123](https://github.com/vectorize-io/hindsight/issues/3123#issuecomment-5148604806)
  (open, filed by another user). No PR from us: two open PRs already implement the two defects
  worth fixing, [#2999](https://github.com/vectorize-io/hindsight/pull/2999) and
  [#3000](https://github.com/vectorize-io/hindsight/pull/3000), both open.

## Root cause

The plugin retains a session by slicing the transcript to the last user turn and sending it. The
user's own prompt goes missing from what is retained, and three separate mechanisms feed that,
with three different origins.

Measured against the plugin's own `content.py`:

```
slice(convo, 1) -> 2 msgs; starts at the tool_result: True
  user's prompt included? False
include_tool_calls=True: full_window=True -> 4 msgs; False -> 2 msgs; prompt survives False? False
include_tool_calls=False: full_window=True -> 3 msgs; False -> 1 msgs; prompt survives False? False
```

**Defect 1 is a regression, and one commit contains both halves of it.** `retain_full_window`
used to be a constant:

```python
# Full session mode: retain all messages, always as full window
retain_full_window = True
```

and `a910fd8a0` (PR #2648, "fix(claude-code): retain session deltas", 2026-07-14) replaced it
with a test on `retention_progress.start_index`. Before that commit `messages_to_retain` was the
whole transcript, so the flag meant "we hold everything, narrow it to the last turn". The same
commit redefined `messages_to_retain` to be the unretained suffix. After it the flag's premise
is gone, and the narrowing is applied to something that was never wide. The line reads as
carried over rather than reconsidered, and it is the only commit that has ever touched it.

**Defect 2 is an unported fix, and the port is the older artifact.** The slice counts a turn
boundary by role, so a synthetic `role: "user"` message carrying a tool result counts as a real
user turn and the window stops at it. The TypeScript sibling does not have this problem, and the
dates say why. The Python `slice_last_turns_by_user_boundary` arrives with the plugin in
`f4390bdc2` (PR #651, 2026-03-23). The equivalent filter on the openclaw side is `3b7d18d47`
(PR #2307, 2026-06-30), three months later, and it states its reasoning in the tree at
`hindsight-integrations/openclaw/src/index.ts:3092-3096`: synthetic tool_result messages
normalised to `role: "user"` "would be counted as real user turns, causing the window to exclude
actual user input from the retained transcript". So the two implementations do not embody two
judgements about tool results. They embody one judgement and its correction, and only one of
them received the correction.

**Defect 3 is deliberate and should be left alone.** `a910fd8a0` added `commit_retention` in two
places behind the same `if retention_progress is not None:` guard, after a successful send and
on the empty-transcript early return. Advancing the checkpoint past a slice that formatted to
nothing is right on its own terms: without it, a window whose every message had a disallowed
role would be reconsidered on every later Stop hook forever. The damage is real, but it arrives
because 1 and 2 hand that branch a slice that is empty only from over-narrowing.

## Invariant violated

**When a commit redefines what a value contains, every predicate derived from the old meaning
still type-checks, still runs, and now answers a question nobody asked.** `start_index == 0` is
a well-formed test on a real field. Nothing in the language, the type system, the tests or the
diff can see that its premise was the old shape of `messages_to_retain`, because a premise is
not a symbol and has no definition site to update. The dangerous configuration is the one here:
**the same commit redefines the value and rewrites the test.** A reviewer reads the new test
against the new intent, the old test never appears in the diff as an open question, and a
constant that was correct by construction becomes a conditional that is wrong on the branch
nobody exercised. Converting a hardcoded `True` into a computed flag looks like tightening. It
is the moment to re-derive the flag from scratch, not to translate it.

**When two implementations of one behaviour disagree, the useful question is not which is right
but which is older.** A divergence carrying a three-month gap is almost never a design fork; it
is a fix that did not travel. The correction is usually the newer artifact and it usually
explains itself, because the person who found the bug wrote down why, so the answer to "should
the Python side filter synthetic tool results?" was already committed in the TypeScript file
months earlier. **A port is a snapshot, and nothing in either tree records that it was one**, so
after a while the copy stops looking like a copy and starts looking like an independent
decision that someone must have had a reason for. Dating the two ends that argument in one
command and turns "which behaviour do we want" into "apply the fix we already made".

**A guard can be correct about its own condition and still be the place where the damage becomes
permanent.** The checkpoint advance is the site where the loss stops being recoverable, so it is
where an investigation naturally stops, and it is the wrong thing to change: removing it fixes
this report and creates a permanent re-retain loop for the case it was written for. That is
worth separating explicitly in any triage, because "where the data is lost" and "what is
defective" are different questions and only the second one is a fix target. Repair the
over-narrowing and the checkpoint advance stops being a data-loss path without being touched.

## Trigger

Any retention pass after the first, since `start_index` is non-zero only once something has been
retained. It reaches a default install: `retain.py:149` reads
`config.get("retainToolCalls", True)`, so the shipped path is the `include_tool_calls=True` one,
4 messages against 2. The prompt is lost on both paths, and the reporter's own 3-versus-1 figures
are the `include_tool_calls=False` path, which a default install does not take.

## Repro

Clean `python:3.12-slim` container at the reporter's sha `b5d8439c8`, importing the plugin's own
`content.py` with provenance printed
(`/w/hindsight-integrations/claude-code/scripts/lib/content.py`) so the measured code is the
repository's and not a paraphrase. A synthetic conversation is built with a real user prompt
followed by an assistant tool call and a synthetic `role: "user"` tool result, then run through
`slice_last_turns_by_user_boundary` and the formatting path under both `include_tool_calls`
settings and both `retain_full_window` values. That is the four-line output above, including the
assistant-only tail returning `(None, 0)`.

The three origins were separated on an unshallowed clone.
`git log -G 'retain_full_window = retention_progress.start_index' --follow -- retain.py` returns
exactly one commit, `a910fd8a0`, and its diff carries the constant it replaced. The port dates
come from the introducing commits of each implementation, `f4390bdc2` for the Python side and
`3b7d18d47` for the openclaw filter, with the rationale comment read in the tree rather than
inferred from the commit message.

**Not verified:** the functions were driven directly and no real session was run with a Stop
hook, so the "second retain onward" framing is read off `retain.py:128-133` rather than observed
end to end. No claim is made about what a live install accumulates over many sessions.

Verified 2026-08-01 at `b5d8439c8`, which is still `main` as of this entry. Reported upstream;
fixes for defects 1 and 2 are open, not merged, at the time of writing.
