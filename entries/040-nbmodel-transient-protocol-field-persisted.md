# A kernel-protocol field written into the document makes the saved notebook invalid

- **Repo:** datalayer/jupyter-nbmodel-client
- **Surface:** `jupyter_nbmodel_client/model.py::save_in_notebook_hook`
- **Class:** message-conversion boundary
- **Fix:** [PR #63](https://github.com/datalayer/jupyter-nbmodel-client/pull/63)
  (in review; reported downstream as
  [jupyter-mcp-server#263](https://github.com/datalayer/jupyter-mcp-server/issues/263))

## Root cause

`save_in_notebook_hook` writes kernel outputs into the shared document verbatim.
For `display_data`, `jupyter_kernel_client.client.output_hook` puts the kernel
protocol's `transient` field into the output, and `transient` is not part of the
nbformat schema. Under RTC the document is autosaved as it stands, so the saved
`.ipynb` stops validating:

```
nbformat.validator.NotebookValidationError: Additional properties are not allowed ('transient' was unexpected)
```

There is already a cleanup, and it cannot help, for two independent reasons.
`KernelClient.execute()` does delete `transient`, but only from its own return
value and only once the execution is over, while the document was written during
it. And pycrdt copies each dict at write time, so that `del` mutates the local
dict and never reaches the copy already in the document.

One detail widens the blast radius: `output_hook` uses `content.get("transient")`,
so the key is written even when the kernel message carries no transient at all,
as `"transient": None`. Every `display_data` output saved through this path is
affected, not only the ones carrying a `display_id`.

## Invariant violated

Anything written into the document must be nbformat valid, because that document
is what gets saved as the notebook. `transient` is kernel messaging protocol, not
notebook content: it may live in the in-process `outputs` list, and never in the
document.

The reusable rule is what a live-syncing store does to the idea of cleanup. With a
CRDT and autosave, "we tidy the data before saving it" stops being a thing you can
do, because there is no moment called saving. The store persists continuously, so
every intermediate state is a state someone can read and write to disk, and the
only safe policy is to never put the field in. A `del` at the end of an operation
protects a return value, not a store. Both of the reasons the existing cleanup
fails are invisible at the `del` site: you cannot see from there that the document
already has a copy, or that the copy was taken by value.

The second lesson is about where to fix it. The obvious move is to stop
`output_hook` from writing `transient` at all, at the source. That was tried as a
counterfactual and it is worse: `output_hook` reads `display_id` back out of the
`outputs` list to match a later `update_display_data`. With `transient` stripped
from `outputs`, the update message matches nothing, `output_hook` returns an empty
index set, and the update is silently dropped, leaving the cell showing a stale
value. The same field is garbage to one consumer and load-bearing to another, so
the fix belongs at the boundary between them, not at the source. "This field is
junk, remove it upstream" is worth measuring before it is worth believing.

## Trigger

Any `display_data` output in an RTC session: a matplotlib figure, anything through
`IPython.display`. No configuration and no rare timing. The notebook on disk is
simply no longer a valid notebook.

## Repro

Python 3.12 container, the branch installed with `pip install -e .`,
`jupyter-kernel-client` at `2dd9bd83`, with `module.__file__` and its sha256
printed for both packages so the runs are pinned to those sources rather than an
installed wheel.

New tests in `jupyter_nbmodel_client/tests/test_model.py`: 3 passed on the branch;
against `main`'s `model.py` with the fix reverted and the tests kept, 2 failed on
`assert "transient" not in ...`, showing `{'output_type': 'display_data', ...,
'transient': None}`. The third test (`update_display_data` updates the output in
place) passes both before and after, which is its purpose: it guards the matching
behavior described above.

Separately from the tests, `save_in_notebook_hook` was driven with an
ipykernel-shaped `display_data` message and `nbformat.validate` run on
`YNotebook.get()`, which is what RTC persists. It fails on `main` with the error
quoted above and passes with the change.

**Not verified:** nothing ran against a live JupyterLab or a real RTC session; the
hook and the document were driven directly, in process. Defect 1 of #263
(executing a cell drops `execution_count` from every code cell) is not addressed
and could not be reproduced without a live RTC server, so a notebook hit by that
defect stays invalid on `execution_count` regardless of this change: this PR fixes
only the `transient` half of that report. `tests/test_client.py` never ran (it
starts a real `jupyter-server` subprocess that was unreachable in the container);
it errors identically on unmodified `main`, so that is the environment and not the
change, but it means CI is being relied on for that leg.

## Fix

Write a sanitized copy into the ycell, and leave `outputs` alone. Leaving
`outputs` intact is required rather than incidental, for the `display_id` matching
reason above.
