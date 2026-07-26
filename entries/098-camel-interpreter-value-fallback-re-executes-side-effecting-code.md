# A "no output, so try evaluating it" fallback re-runs the same code, so a silent statement executes twice

- **Repo:** camel-ai/camel
- **Surface:** `camel/interpreters/internal_python_interpreter.py`, the `unsafe_mode`
  branch: `exec(code)` at `:194`, `if not result:` at `:198`, `str(eval(code))` at `:200`
  (line numbers at the verified commit)
- **Class:** tool side-effect scope
- **Report:** reproduced publicly on
  [issue #4180](https://github.com/camel-ai/camel/issues/4180#issuecomment-4992650786),
  fixed upstream by
  [PR #4197](https://github.com/camel-ai/camel/pull/4197) (merged 2026-07-20, "execute
  unsafe Python expressions only once"). Triage only, no PR from us: the reporter was
  ready to fix it and this lane was under a standing throttle on camel PR volume.

## Root cause

The interpreter wants to return a useful string whether it was handed a statement (whose
value is `None`, and whose visible effect is printed output) or an expression (whose value
is the answer). It decides which it received by running the code and inspecting what came
back:

```python
output_buffer = io.StringIO()
with contextlib.redirect_stdout(output_buffer):
    exec(code, self.action_space)
result = output_buffer.getvalue()

# If no output was captured, try to evaluate the code
if not result:
    try:
        result = str(eval(code, self.action_space))
    except (SyntaxError, NameError):
        result = ""
```

`exec` has already run the code once. When it printed nothing, the fallback runs *the same
source string* a second time to obtain a value. For pure expressions this is invisible;
for anything with a side effect it is not. `log.append('sent')` appends twice, and because
`eval` returns `None`, the caller is told `Executed Results: None`, so no observable part
of the response indicates a second run.

The guard is `if not result`, on captured stdout, which conflates "produced no output" with
"is an expression". A statement that prints fills the buffer, skips the branch, and
correctly runs once. That is why the reporter's control case behaves differently:
`print(d2.append(2))` yields `[2]`, a single append.

Both settings have to be off-default, which bounds the exposure and is worth stating with
the defect. `CodeExecutionToolkit` defaults to `sandbox="subprocess"` and builds a
`SubprocessInterpreter`; only `sandbox="internal_python"` constructs the affected class
(`camel/toolkits/code_execution.py:95-99`), and `unsafe_mode=True` is required to reach the
branch at all.

## Invariant violated

Code is executed once per request to execute it. A fallback that exists to obtain a
*value* must re-read, re-parse or re-interpret the input, never re-run it, because
execution is not idempotent in general and the fallback cannot know whether this
particular input is.

The specific antipattern is using an *effect* as a type test. Deciding "statement or
expression" by observing what running the code produced requires running it, so the
decision necessarily comes after the irreversible part, and the only recovery available is
another irreversible part. The information wanted is static and available for free:
`compile(code, "<string>", "eval")` succeeds exactly for expressions and raises
`SyntaxError` for statements, so the branch can be chosen before anything runs, and each
input is then executed once by the appropriate one of the two. That is the shape the
upstream fix takes.

The failure mode is also biased toward silence in a way worth generalizing. The duplicate
is invisible in the return value (the second run's value replaces nothing the caller can
compare against), invisible in the output (the branch is only taken when there was no
output), and invisible in the *common* case, because the inputs that trigger it are
precisely the ones that print nothing. In an agent loop, that means the model's own
observation of its tool call is consistent with a single execution, so neither the model nor
its evaluator can detect that the world moved twice. A tool whose real effect exceeds what
it reports removes the agent's ability to reason about its own actions, which is a
correctness problem for the agent framework and not only for the interpreter.

The general check for any "try A, and if that did not work try B" over an effectful
operation: is B a retry of A, or a different way of reading A's result? If A can change
state, the fallback must be the second thing, and if the code cannot be written that way,
the branch must be decided before A runs.

## Trigger

`CodeExecutionToolkit(sandbox="internal_python", unsafe_mode=True)`, or
`InternalPythonInterpreter(unsafe_mode=True)` directly, given a single expression that
prints nothing and has a side effect. Mutating method calls (`list.append`, `dict.update`,
`set.add`), assignments through a helper, and any function call whose result is not printed
all qualify. Default configurations are unaffected.

## Repro

Clean `python:3.11-slim`, `pip install -e .`, no network, master HEAD
`0b9d9862b863af13d54c971fdff0c2c7e2f2ae01`, `__file__` provenance printed from
`/src/camel/...`. The reporter's direct-interpreter case gives `[1, 1]`. Through the
toolkit, which is the path LLM-generated code actually takes:

```python
tk = CodeExecutionToolkit(sandbox="internal_python", unsafe_mode=True, require_confirm=False)
tk.interpreter.action_space["log"] = []
tk.execute_code("log.append('sent')")
```

```
interpreter type: InternalPythonInterpreter
action_space['log'] = ['sent', 'sent']
returned: "Executed the code below:\n```python\nlog.append('sent')\n```\n> Executed Results:\nNone"
```

Control, verifying the guard is stdout and not the presence of a side effect:
`print(d2.append(2))` gives `[2]`, one run.

Verified 2026-07-16 at HEAD `0b9d9862`. Fixed upstream: at HEAD `ec48f997` the branch is
chosen statically by `compile(code, "<string>", "eval")`, with `exec` reached only from the
`SyntaxError` handler, so each input runs exactly once.
