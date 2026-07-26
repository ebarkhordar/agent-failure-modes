# A recursive lookup returns the first subtree's result unconditionally, so a key in a sibling subtree reads as absent

- **Repo:** deepspeedai/DeepSpeed
- **Surface:** `deepspeed/autotuning/utils.py`, `get_val_by_key` at `:133-139`, against its
  paired setter `set_val_by_key` at `:142-147` (line numbers at the verified commit)
- **Class:** silent wrong answer from a search helper
- **Report:** [issue #8176](https://github.com/deepspeedai/DeepSpeed/issues/8176), fixed by
  [PR #8177](https://github.com/deepspeedai/DeepSpeed/pull/8177) (merged 2026-07-26 by
  tohtana)

## Root cause

The helper searches a nested config dict for a key at any depth:

```python
def get_val_by_key(d: dict, k):
    if k in d.keys():
        return d[k]
    for v in d.values():
        if isinstance(v, dict):
            return get_val_by_key(v, k)
    return None
```

The loop over `d.values()` is written as if it were iterating over candidates, but the
`return` inside it is unconditional. The first dict-valued entry consumes the whole loop:
whatever that subtree yields, including `None`, is returned as the answer for the entire
document. Every sibling subtree after the first is unreachable.

So the function is not "search the tree"; it is "search the leftmost path". Whether a key
is found depends on where it sits in Python's dict insertion order relative to the other
subdicts, which no caller controls or thinks about.

The repo contradicts itself two lines down. `set_val_by_key`, the inverse operation on the
same data, recurses into every subdict with no early return, so it reaches keys the getter
reports as absent. A deliberate first-match-only search would not have been written to
return `None` out of a subtree without checking the rest, and it would not have been paired
with a setter that does check the rest.

## Invariant violated

A recursive search returns a negative result only after exhausting the search space. The
recursive call's `None` is not the function's answer, it is one branch declining to
answer, and only the caller that has run out of branches may convert it into `None`.

The concrete shape to look for: a `return f(child)` directly inside a loop over children.
That is a loop that cannot iterate. It reads as a search because the loop is there, and it
behaves as a single lookup because the `return` is unguarded. The repair is one line,
`r = f(child); if r is not None: return r`, and the tell that it is needed is a
sentinel-returning recursive call whose result is not inspected.

Also worth generalizing: a get/set pair over the same structure is a free oracle. The two
must agree about which keys exist, so any asymmetry in how they traverse is a bug in one of
them, and the answer to "which one" is usually whichever one takes a shortcut.

## Trigger

Any config dict where the sought key lives in a subdict that is not the first dict-valued
entry. Consumed at `deepspeed/autotuning/scheduler.py:98`:

```python
nval = get_val_by_key(exp, key)
if nval and nval != val:
    ...  # overwrite the CLI arg with the tuned value
```

A wrong `None` fails the `if nval` guard, so the argument is silently left at the user or
default value and the autotuning experiment runs without the value being tuned. There is no
error and no log line; the run simply measures the wrong configuration.

The severity is bounded by insertion order, and that bound is worth stating rather than
glossing: if the key's subdict happens to be visited first, the lookup is correct, so the
same code path is right or wrong depending on how the config dict was built.

## Repro

Standalone, CPU only, function taken verbatim from HEAD `d3265209`, `__file__` provenance
printed from the checkout:

```python
exp = {"optimizer": {"type": "Adam"},
       "zero_optimization": {"offload_optimizer": {"device": "cpu"}}}

get_val_by_key(exp, "device")        # -> None      (bug; "cpu" is present)
set_val_by_key(exp, "device", "nvme")  # -> reaches the field and rewrites it
```

The setter reaching a field the getter cannot see is the asymmetry described above.

## Fix

[PR #8177](https://github.com/deepspeedai/DeepSpeed/pull/8177): keep recursing when a
subtree returns `None`, and a unit test with two sibling subdicts where the key is in the
second.

Not established, and stated as such in the PR: no end to end autotuning session was run, so
the claim is about the helper and about the static call at `scheduler.py:98`, not a measured
production tuning failure.
