# An override of a private stdlib method is pinned to the arity that method had when it was written, so a CPython point release turns it into a TypeError

- **Repo:** huggingface/datasets
- **Surface:** `src/datasets/utils/_dill.py:73` (the `_batch_setitems` override) and `:82`
  (its `super()` call)
- **Class:** dependency & runtime contract drift
- **Report:** measured publicly on
  [issue #8373](https://github.com/huggingface/datasets/issues/8373#issuecomment-5152899500)
  (open, filed by another user). No PR: the defect is already repaired at HEAD by
  [PR #7817](https://github.com/huggingface/datasets/pull/7817) and shipped in 4.4.0. What
  the thread needed was the version boundary, not a patch.

## Root cause

`datasets` subclasses dill's `Pickler` and overrides `_batch_setitems` so that dict keys are
sorted before being written, which is what makes a fingerprint of a dict independent of its
key order and therefore makes the cache hit. That method belongs to CPython's `pickle`
module, it is private, and in 3.14 it takes an argument it did not take before. The override
declared two parameters, `save_dict` now calls it with three, and the call fails before any
of the sorting logic runs.

Measured on Python 3.14.6, one virtualenv per version:

```
datasets==4.3.0 (dill 0.4.0)  -> TypeError: Pickler._batch_setitems() takes 2 positional
                                 arguments but 3 were given
                                 raised from /usr/local/lib/python3.14/pickle.py:1058, save_dict
datasets==4.4.0 (dill 0.4.0)  -> load_dataset OK, 3 rows
datasets==5.0.1 (dill 0.4.1)  -> load_dataset OK, 3 rows
main @ b7cb10b (dill 0.4.1)   -> load_dataset OK, 3 rows
```

The repair at HEAD is the shape the situation calls for:

```python
def _batch_setitems(self, items, *args, **kwargs):
    ...
    return super()._batch_setitems(items, *args, **kwargs)
```

The override takes what it came for, forwards everything else untouched, and stops caring how
many arguments the base method grows.

The thread had stalled on two claims that both sounded like evidence and neither of which
settles the question: that the fix already landed in #7817, and that CI was red on the commit
that landed it. Both are true. The check runs on that merge commit include four failures, and
none of them exercises this path, so the state of that run says nothing about whether the code
is correct. The version matrix answers it directly, and one more detail closes the loop: the
patch the reporter proposed quotes the body `_batch_setitems` had before 4.4.0, which is
evidence about their installed copy rather than about the project.

## Invariant violated

**Overriding a method whose name starts with an underscore inherits its signature as a
contract that nobody promised to keep.** A public API changes on a deprecation cycle with a
release note. A private one changes when its owner finds it convenient, which is the meaning
of the underscore, and a point release of the interpreter is entirely within its rights to add
a parameter. The subclass has no notification channel: nothing imports the signature, no type
checker sees across that boundary, and the first report is a user's traceback.

**The trap is that the functionality is legitimate and the only mechanism available is
unsupported.** Deterministic key ordering during pickling is a reasonable thing to want, and
`pickle` offers no public hook for it, so reaching into `_batch_setitems` is not carelessness,
it is the only door. That combination is worth recognising as a class of its own: a real
requirement, no supported extension point, and a private method that happens to sit exactly
where the requirement lives. The code is right to be there and is permanently exposed.

**When an override exists to intercept rather than to reimplement, forward everything you did
not come for.** `*args, **kwargs` on the parameter list and on the `super()` call costs
nothing and converts every future signature change from a crash into a passthrough. A fixed
parameter list, by contrast, encodes a snapshot of the base method taken on the day the
override was written, and each new argument upstream adds turns that snapshot into a
TypeError. The rule generalises past pickle to every wrapper, decorator and monkeypatch that
means to add a step and then delegate.

**In a stalled thread the useful move is usually a boundary, not an argument.** "It is fixed"
and "CI was red" are both statements about the repository, and the reporter is asking a
question about their machine. Running the matrix converts an exchange of assertions into a
single fact, which version stops failing, and that also names the upgrade that resolves it.
Related: what a reporter quotes identifies what they are running. A proposed patch written
against a body of code that no longer exists upstream dates their install more reliably than
asking them to report a version, because it is not subject to their reporting the wrong
environment.

## Trigger

Python 3.14 with `datasets` below 4.4.0. The failure lands on any `load_dataset` that
fingerprints a dict, so it presents as the library being broken on 3.14 rather than as
anything to do with pickling, and it disappears on upgrade, which is why the thread turned
into a disagreement about whether it had been fixed rather than about what it was.

## Repro

Clean `python:3.14-slim` containers, one virtualenv per version under test, with no network
beyond PyPI, running the same `load_dataset` call against each of 4.3.0, 4.4.0, 5.0.1 and an
editable install of `main` at `b7cb10b`. That produced the matrix above. The first release
carrying the fix comes from `git tag --contains` on the fixing commit rather than from the
changelog. The CI figures come from the check runs on that commit, counted rather than
characterised.

**Not verified:** the traceback measured here is the mirror of the reporter's. Theirs reaches
CPython's own `_batch_setitems` and ours reaches the `datasets` override, which was stated
plainly in the public comment instead of being claimed as an identical reproduction. No claim
is made about other subclasses of `Pickler` in the wider ecosystem, which were not surveyed.

Verified 2026-08-01 against `b7cb10b`. Fixed upstream and released; the issue is still open at
the time of writing because the version question, not the code, is what remained.
