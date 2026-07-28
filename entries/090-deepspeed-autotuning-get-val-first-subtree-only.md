# A recursive dict lookup returns its first subtree's result unconditionally, so a key in any later sibling subtree reads as absent

- **Repo:** deepspeedai/DeepSpeed
- **Surface:** `deepspeed/autotuning/utils.py`, `get_val_by_key` (consumed by
  `deepspeed/autotuning/scheduler.py`)
- **Class:** recursive traversal completeness
- **Fix:** [PR #8177](https://github.com/deepspeedai/DeepSpeed/pull/8177) (merged
  2026-07-26 by tohtana; issue
  [#8176](https://github.com/deepspeedai/DeepSpeed/issues/8176))

## Root cause

`get_val_by_key` searches a nested config dict for a key. On a miss at the current
level it recurses, but it returns the result of the recursive call on the *first*
dict-valued entry unconditionally:

```python
def get_val_by_key(d: dict, k):
    if k in d:
        return d[k]
    for v in d.values():
        if isinstance(v, dict):
            return get_val_by_key(v, k)   # commits to the first subdict, even on None
    return None
```

The loop is written as if it were iterating over candidates, but the `return` inside it
is unguarded, so the first dict-valued entry consumes the whole loop. The function is not
"search the tree", it is "search the leftmost path".

If the target key lives under a later sibling subdict, the function has already
returned `None` out of the first subtree and never looks at the rest. A correct
search tests each recursive call and only returns a hit, falling through on a miss:
`r = get_val_by_key(v, key); if r is not None: return r`. The paired mutator
`set_val_by_key` in the same file recurses into *all* subdicts with no early return,
so the getter is plainly meant to be its inverse; the asymmetry between the two is
the frame: set reaches the key, get does not. `git log -G 'get_val_by_key' --follow`
shows the function untouched since its 2021 introduction (`9caa74e5`, #1554): born
with the bug, never fixed, never reverted.

## Invariant violated

A recursive search that returns `rec(first_child)` directly is not a search: it
commits to the first subtree and cannot see its siblings, so the answer depends on
iteration order over the children rather than on where the key actually is. To find a
key that may live in any subtree, each recursive descent must be tested and only a
found value propagated, with a miss falling through to the next sibling. When a data
structure has a paired reader and writer, they must agree on traversal completeness;
a writer that visits every subtree and a reader that stops at the first are
inconsistent, and the writer's behavior is the contract the reader is failing to
meet.

The concrete shape to grep for is a `return f(child)` sitting directly inside a loop over
children: a loop that cannot iterate. It reads as a search because the loop is there and
behaves as a single lookup because the `return` is unguarded. The tell that the repair is
needed is a sentinel-returning recursive call whose result is never inspected.

## Trigger

`get_val_by_key(config, key)` where `key`'s containing subdict is not the first
dict-valued entry encountered in iteration order. In autotuning, `scheduler.py`
injects a tuned value via `nval = get_val_by_key(exp, key)`; when the lookup wrongly
returns `None`, the guarding `if nval and ...` is skipped and the CLI arg is not
overwritten, so (*conditional on the tuned key's subdict not being searched first*)
the experiment runs with the user/default value instead of the value being tuned.

## Repro

CPU-only, at HEAD `d3265209`, the verbatim function against a two-subdict config:
`exp = {'optimizer': {'type': 'Adam'}, 'zero_optimization': {'offload_optimizer':
{'device': 'cpu'}}}`. `get_val_by_key(exp, 'device')` returns `None` (bug); a
corrected sibling-continuing search returns `'cpu'`; `set_val_by_key(exp, 'device',
'nvme')` does reach and mutate the field, confirming the get/set asymmetry. This is
an existence proof at the helper level; whether a real autotuning session hits an
order that bites is left to the config's dict insertion order and is not claimed
here. The fix rewrites the search to continue across siblings and adds a unit test
with the key in the second subdict.
