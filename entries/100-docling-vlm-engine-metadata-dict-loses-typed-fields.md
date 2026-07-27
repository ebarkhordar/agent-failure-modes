# Four typed constructor calls collapsed into one mapper over an untyped `metadata` dict, so which fields get filled becomes a per-producer convention nothing checks

- **Repo:** docling-project/docling
- **Surface:** `docling/models/stages/vlm_convert/vlm_convert_model.py`,
  `_prediction_from_engine_output`, against the four engines under
  `docling/models/inference_engines/vlm/` that feed it and the four legacy models under
  `docling/models/vlm_pipeline_models/` it replaced
- **Class:** streaming & usage accounting
- **Report:** reproduced publicly on
  [issue #3869](https://github.com/docling-project/docling/issues/3869#issuecomment-5086817635).
  No PR from us: the reported half was already fixed upstream by
  [PR #3818](https://github.com/docling-project/docling/pull/3818) (merged 2026-07-23,
  shipped in v2.115.0) when we reproduced it, so the entry records the mechanism and what
  the fix does not reach.

## Root cause

Docling has two ways to run a vision model. On the legacy path each model builds its own
result object directly, and all four do it the same way, as a keyword call naming the
fields:

```python
yield VlmPrediction(
    text=...,
    generated_tokens=tokens,
    generation_time=...,
    input_prompt=input_prompt,
)
```

The newer preset path inverts this. Each engine returns a `VlmEngineOutput` carrying a
free-form `metadata: dict`, and one shared mapper turns that into the same
`VlmPrediction`. In v2.113.0 the mapper read nothing out of the dict at all:

```python
def _prediction_from_engine_output(output: VlmEngineOutput) -> VlmPrediction:
    stop_reason = VlmStopReason.UNSPECIFIED
    if output.stop_reason in _VLM_STOP_REASON_VALUES:
        stop_reason = VlmStopReason(output.stop_reason)

    return VlmPrediction(text=output.text, stop_reason=stop_reason)
```

Every engine's token counts and timings were computed, packed into `metadata`, handed
across, and dropped on the floor. Nothing raised, because each missing field has a model
default: `num_tokens=None`, `usage=None`, `generation_time=-1`, `generated_tokens=[]`,
`input_prompt=None`. A caller reading `prediction.num_tokens` gets `None`, which is
indistinguishable from a model that reported no usage.

PR #3818 repaired the mapper, and that is where the interesting part starts. The reason
the field could go missing is not the three lines; it is that the field list stopped being
a signature. On the legacy path, adding a field to `VlmPrediction` puts it in front of
four call sites a reader and a type checker can enumerate. On the engine path it is a
string key in a dict each engine populates independently, so completeness is a convention,
and conventions are checked by nobody.

That prediction is visible in the fixed version. `_prediction_from_engine_output` now
reads three keys, and in v2.115.0 as installed the four engines fill them like this:

| engine | `num_tokens` | `usage` |
| --- | --- | --- |
| `api_openai_compatible_engine.py:195` | set | set |
| `transformers_engine.py:497` | set | absent |
| `vllm_engine.py:344` | set | absent |
| `mlx_engine.py:271` | absent | absent |

`mlx_engine` puts only `generation_time` and `model` in its dict, so token counts are still
`None` there. The `usage` blanks look deliberate, since no legacy local model set that
field either.

The other two fields are worse off, because the mapper does not read them at all.
`generated_tokens` and `input_prompt` appear nowhere under
`docling/models/inference_engines/` or `docling/models/stages/`, so on the preset path they
sit at their defaults for every engine. That takes a documented option with it:
`track_input_prompt` is declared twice in `pipeline_options_vlm_model.py` (`:335`, `:442`)
and read at exactly four sites, all four of them in `models/vlm_pipeline_models/`. Six
references package-wide, no dynamic lookup on the engine path. Set it on a preset pipeline
and it changes nothing.

## Invariant violated

When a typed constructor call is replaced by a mapper over an untyped payload, the type
system stops enforcing the field list and no replacement enforcement appears by itself. The
signature was doing work that is easy to miss precisely because it was doing it silently:
it named the fields, it put every producer in front of the same names, and it made adding
a field a change that shows up at each site.

The consequence to expect, and the one to go looking for, is that a field's population
becomes a per-producer choice with no error path. So a fan-in mapper needs its contract
stated somewhere that fails loudly: a typed intermediate rather than `dict[str, Any]`, or
a test that asserts each producer's payload carries the required keys. The default-valued
target field is what turns a missing key into silence, and it is also why the gap survives
review: every value is legal, every read succeeds, and the wrong answer has the same shape
as a legitimate "not reported".

The second, cheaper rule: when a refactor moves a path, the fields the old path filled are
the checklist for the new one. `git log` for the deleted call sites and diff their keyword
lists against what the replacement produces. Here that comparison is four constructor calls
against four dict literals, and it finds both residuals in a minute, including the option
that is now inert.

## Trigger

Any conversion driven through the preset or engine VLM runtime, which is the path
`ApiVlmEngine` and its siblings serve. Before v2.115.0 the loss was total: every field
except `text` and `stop_reason` stayed at its default regardless of engine. After it, on
`mlx_engine` token counts are still lost, and on all four engines `generated_tokens`,
`input_prompt` and therefore `track_input_prompt` are inert. Downstream this reads as a
model that returned no usage, so cost accounting and any per-page token budget built on
`VlmPrediction.num_tokens` silently sees zero activity rather than an error.

## Repro

Two runs of one script in `python:3.12-slim`, differing only in the installed
`docling-slim` version, with `__file__` provenance printed from the site-packages copy
(`/usr/local/lib/python3.12/site-packages/docling/models/stages/vlm_convert/vlm_convert_model.py`).
The script calls `_prediction_from_engine_output` with the metadata dict that
`api_openai_compatible_engine` builds:

```python
meta = {"num_tokens": 1234,
        "usage": {"prompt_tokens": 1000, "completion_tokens": 234, "total_tokens": 1234},
        "generation_time": 1.25}
p = _prediction_from_engine_output(VlmEngineOutput(text="hello", metadata=meta))
```

```
2.113.0 -> num_tokens=None  usage=None  generation_time=-1
2.115.0 -> num_tokens=1234  usage={'prompt_tokens': 1000, ...}  generation_time=1.25
```

The residuals in the table above were read from each engine's own `metadata={...}` literal
in the same installed 2.115.0 tree, and the `track_input_prompt` reader count is an
uncapped search of the whole package.

**Scope of what was measured.** No VLM was run and no conversion was driven against a live
endpoint. Every claim here is about the engine-output to prediction mapping, which is where
the reported drop was, plus a source-level enumeration of what the four producers put in
the dict. `mlx_engine` requires Apple silicon and was not executed; the statement about it
is what its `metadata` literal contains, run through the mapper.
