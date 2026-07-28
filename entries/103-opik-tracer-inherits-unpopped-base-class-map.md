# A subclass fixes the per-run maps it owns, and keeps growing through one it inherits from a base class with no removal site

- **Repo:** comet-ml/opik
- **Surface:** `OpikTracer` (LangChain integration) and its `RunStateStore`, against
  `order_map`, written by `_TracerCore._start_trace` at
  `langchain_core/tracers/core.py:148`
- **Class:** state lifecycle & unbounded growth
- **Report:** reproduced publicly on
  [issue #7516](https://github.com/comet-ml/opik/issues/7516#issuecomment-5098434143) (open).
  The reported leak is already fixed on `main` by
  [PR #7566](https://github.com/comet-ml/opik/pull/7566) (merged 2026-07-22, `dad2d502`), so
  no PR from us; the entry records the mechanism and the residual that the fix does not reach.

## Root cause

`OpikTracer` keeps per-run bookkeeping so it can attach each span to the right trace. In
2.1.31 those maps were written on every run and released nowhere, so a long-lived tracer grew
for the life of the process and pinned the prompt and completion payloads with it. The
reporter's repro measures exactly that. On `main` the bookkeeping moved into a `RunStateStore`
and is released in the root run's end handler. Clean `python:3.10-slim`,
`langchain-core==1.4.9`, 50 invocations of one long-lived tracer:

```
opik 2.1.31    _span_data_map=100  _created_traces_data_map=150
               _skipped_langgraph_root_run_ids=50  _langgraph_parent_span_ids=50
               retained_payload_bytes=13123

main e72da637  _span_data_by_run_id=0  _trace_data_by_run_id=0
               _skipped_langgraph_root_run_ids=0
               retained_payload_bytes=0
```

The interesting part is what did not go to zero. Counting every dict, list and set on the
tracer object at 50, 200 and 500 invocations:

```
_created_traces                              50    200    500
order_map (inherited)                       150    600   1500
run_map                                       0      0      0
_run_state._span_data_by_run_id               0      0      0
_run_state._trace_data_by_run_id              0      0      0
_run_state._skipped_langgraph_root_run_ids    0      0      0
_external_run_ids                             0      0      0
```

`_created_traces` is deliberate: #7566 keeps it to back the public `created_traces()`. It
holds `Trace` objects carrying no payloads, 277 bytes per entry as measured here, so it is
minor next to what was fixed but still unbounded in a process that never rebuilds its tracer.

`order_map` is not opik's at all. `_TracerCore._start_trace` writes `self.order_map[run.id]`
at `langchain_core/tracers/core.py:148`, and that file contains no site that removes from it,
whereas the adjacent `run_map` *is* popped in `_end_trace`. A stock `BaseTracer` subclass with
opik absent from the process grows identically, 150/600/1500 on the same langchain-core, which
is what establishes the ownership. It is also the larger share of what remains: roughly 750
bytes per invocation between the `order_map` entry and the dotted-order strings it holds,
against about 168 bytes still retained inside opik.

## Invariant violated

A map is bounded by its *removal* site, not by anything visible where it is written. Every
unbounded map looks correct at the write site, because writing one entry per run is precisely
what a per-run map is for. So an audit that reads the code that grows is reading the half that
cannot be wrong, and only an audit that pairs each write with a pop, on the lifecycle event
that ends the run, can tell a cache from a leak.

The second rule is about where that pairing is allowed to live. A subclass can only fix the
containers it declares. When a base class writes per-item state on behalf of the subclass's
traffic, the growth is attributed to the subclass by every external measurement (it is on the
subclass's object, it scales with the subclass's calls, and it appears in the subclass's
memory profile) while the only possible repair sits in a dependency. Both sides then audit
honestly and find nothing: the subclass because the map is not its, the base class because
nobody reported a leak against it. That is how a fixed leak stays visibly unfixed, and it is
why the control run matters more than the measurement it accompanies. Reproducing the growth
with the suspected library *absent from the process* is the only step that converts a
correlation into an owner.

The third is the sharpest and the cheapest to act on. Adjacency to a correctly managed map is
the strongest false reassurance available. `run_map` and `order_map` are written in the same
method of the same class on the same lifecycle, and exactly one of them is popped. A reviewer
who checks that "this class cleans up its run state" checks the first name and stops, because
the pattern is present. Bound the check to the specific container, never to the class.

## Trigger

The long-lived-tracer pattern: one `OpikTracer` instance reused across many invocations, as
opposed to constructing one per request. Before #7566 the whole per-run state accumulated,
payloads included. After it, memory still grows at roughly 750 bytes per invocation from
langchain-core's `order_map` plus a further 277 bytes per entry in `_created_traces`. A
process that rebuilds its tracer, or one that is short-lived, never accumulates enough for
either to matter.

## Repro

Clean `python:3.10-slim` with `langchain-core==1.4.9`, comparing `opik==2.1.31` from PyPI
against `main` at `e72da637`, driving one tracer through 50 invocations and reading the
container sizes directly off the tracer object. 2.1.31 reproduces the reporter's numbers
exactly, which is what pins the comparison. The residual table is the same measurement at 50,
200 and 500 invocations. The ownership claim for `order_map` is a separate run: a stock
`BaseTracer` subclass, with opik not installed in the process, grows 150/600/1500 at the same
three points.

**Scope of what was measured.** Container sizes and retained payload bytes on the tracer
object, not process RSS, so this is a statement about what the tracer holds rather than about
total memory behaviour under a real workload. The byte figures are from this container and
this langchain-core version; they are orders of magnitude, not a contract.
