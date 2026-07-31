# A weight-loading method is defined on one of four sibling wrappers, and the dispatch table still routes the other three into the caller that requires it

- **Repo:** kvcache-ai/ktransformers (the calling half lives in the vendored
  `kvcache-ai/sglang` fork at `third_party/sglang`)
- **Surface:** `kt-kernel/python/utils/amx.py:549` (`NativeMoEWrapper`, the only definition of
  `submit_write_weight_scale_to_buffer`), against the backend selection at
  `kt-kernel/python/experts.py:340-356` and the four call sites in
  `third_party/sglang/python/sglang/srt/layers/moe/kt_ep_wrapper.py:1004`, `:1171`, `:1419`,
  `:1571`
- **Class:** initialization & control flow
- **Report:** triaged publicly on
  [issue #2113](https://github.com/kvcache-ai/ktransformers/issues/2113#issuecomment-5146208615)
  (open, filed by another user running MiniMax M2.7). No PR from us: the repair is either a
  startup guard in the sglang fork or two new methods on two kernel wrappers wired to C++
  kernels that may not implement the underlying write-buffer task at all, and neither is
  verifiable without the GPU host the path needs, so rule 2 forbids the behavioural claim a PR
  would have to make.

## Root cause

`submit_write_weight_scale_to_buffer` and its `sync_` partner are defined on exactly one class
in the tree, `NativeMoEWrapper` (`amx.py:549`). They are absent from the other three concrete
backends and from the base:

| class | defined at | has the method |
|---|---|---|
| `NativeMoEWrapper` | `amx.py:549` | yes |
| `AMXMoEWrapper` | `amx.py:248` | no |
| `LlamafileMoEWrapper` | `llamafile.py:21` | no |
| `GeneralMoEWrapper` | `moe_kernel.py:29` | no |
| `BaseMoEWrapper` (ABC) | `experts_base.py:227` | no |

All four concrete classes derive from `BaseMoEWrapper`, and `_create_inference_wrapper`
(`experts.py:340-356`) picks between them on the `--kt-method` string alone: `AMXINT4`/`AMXINT8`
to `AMXMoEWrapper`, `LLAMAFILE` to `LlamafileMoEWrapper`, `MOE_INT4`/`MOE_INT8` to
`GeneralMoEWrapper`, and the eight remaining methods (`RAWINT4`, `FP8`, `BF16`,
`FP8_PERCHANNEL`, `GPTQ_INT4`, `SYCL_GPTQ_INT4`, `MXFP4`, `MXFP8`) to `NativeMoEWrapper`.

The consumer does not know about that split. `SharedFullContext.load()`
(`kt_ep_wrapper.py:1662-1690`) selects one of six `_prepare_weight_*` helpers by quantization
flag, and four of them contain a `submit_write_expert()` that calls the method on the `wrapper`
they were handed (`mxfp4` and `mxfp8` delegate into the fp8 helper, so all six branches reach
one). That `wrapper` is `KTMoEWrapper(...)` built at `:2661`, and `KTMoEWrapper.__new__`
(`experts.py:120`) returns `_create_inference_wrapper(...)`, which is to say one of the four
classes above. Nothing between the dispatch and the call re-checks what came back.

So a run started with `--kt-method LLAMAFILE` or `AMXINT4` reaches, at weight-load time, a call
that only the `NativeMoEWrapper` branch can satisfy, and raises
`AttributeError: 'LlamafileMoEWrapper' object has no attribute
'submit_write_weight_scale_to_buffer'`.

The reporter's own hypothesis was that sglang is not optimised for pure AVX2 and that some
other sglang version would support it. The measurement contradicts that: `AMXMoEWrapper` is
missing the same two methods, so an AMX-capable host on `--kt-method AMXINT4` fails identically.
The determinant is the quantisation method, not the CPU instruction set, and a version
downgrade is unlikely to help.

## Invariant violated

**An interface that exists only as a set of call sites is checked nowhere, so adding a
capability to one implementation silently makes every sibling non-conforming.** `BaseMoEWrapper`
is an `ABC` and would have made this a construction-time error for free by declaring the method
abstract, or a clean `NotImplementedError` by declaring it concrete. It declares neither, so the
contract that `_prepare_weight_*` depends on is written down in no class, no protocol and no
type annotation, and exists only as the fact that four expressions happen to name it. Under that
arrangement the set of classes that must implement a method is not a fact anyone can look up: it
is the transitive image of a dispatch table over a set of call sites, recomputed by hand every
time someone changes either end.

**The failure is deferred to dispatch, so the blast radius is decided by a config string rather
than by the code that changed.** Whoever added these two methods added them beside the kernel
that needed them, and that commit is not obviously incomplete when read: the class it touched
does implement what it calls. What made it a defect is a routing table in a different package
that keeps sending three other classes into the same consumer. This is why the useful question
after adding a method to one member of a substitutable family is not "is my class correct" but
"who else can arrive here", and the second question is answerable only from the dispatch site.

**A base class that exists but declares nothing is worse than no base class, because it
advertises a contract it does not hold.** `BaseMoEWrapper` sits at the top of all four, so its
presence makes the four look interchangeable to a caller, which is exactly the belief
`_prepare_weight_*` acts on. An `ABC` whose abstract surface is smaller than the surface its
consumers actually use converts a compile-time-shaped guarantee into a runtime `AttributeError`
at the worst moment, after model weights are already being loaded.

## Trigger

An inference run (`mode` other than `sft`) with `--kt-method` set to `LLAMAFILE`, `AMXINT4`,
`AMXINT8`, `MOE_INT4` or `MOE_INT8`, reaching the full-context path: `_build_full_context` is
entered only when `gpu_prefill_token_threshold > 0` and the prefill batch is at least that
(`kt_ep_wrapper.py:2514`, `:2955-2956`). The corresponding `server_args` field defaults to
`None` (`server_args.py:732`), so the gate is off unless the operator sets it, which is why this
is a specific-configuration crash rather than a universally broken backend. The SFT path is out
of scope: `mode="sft"` routes to `AMXSFTMoEWrapper` (`experts.py:430`), a different family.

## Repro

Not run behaviourally. The path needs an AVX2 host with 2x4090 48G, 256G DDR4 and the
MiniMax-M2.7 GGUF, and this machine has no GPU. The reporter's traceback on issue #2113 is the
runtime evidence that the lookup fails; what is measured here is the structural claim that it
must, for three of the four backends.

Measured in a clean `python:3.12-slim` container with the clones mounted read-only, at
ktransformers `a8062bf` with the `third_party/sglang` submodule at its pinned
`1e098a77`. Class members were enumerated by parsing (`ast`, walking each `ClassDef` body), not
by grep, because the question is "what is the complete set of methods on this class" and a text
search cannot answer that:

```
AMXMoEWrapper          amx.py:248     submit_write_weight_scale_to_buffer = False
NativeMoEWrapper       amx.py:549     submit_write_weight_scale_to_buffer = True
LlamafileMoEWrapper    llamafile.py:21    submit_write_weight_scale_to_buffer = False
GeneralMoEWrapper      moe_kernel.py:29   submit_write_weight_scale_to_buffer = False
BaseMoEWrapper         experts_base.py:227 submit_write_weight_scale_to_buffer = False
```

File provenance for that run, `sha256[:16]`: `amx a3b0434216d8d0f2`, `llamafile
ade69519c7f9c07f`, `moe_kernel 824181b2fea11962`, `experts_base e310c2d5e4c9aafb`, `experts
8d07f35f89b219bd`, `kt_ep_wrapper b1f11e6f8086b3b3`.

The static enumeration is only sound if attribute lookup cannot be intercepted, so that was
checked rather than assumed: none of `amx.py`, `llamafile.py`, `moe_kernel.py`,
`experts_base.py` or `experts.py` defines `__getattr__` or `__getattribute__`. The tree does
contain one `__getattr__`, at `kt_ep_wrapper.py:3288`, and it belongs to `KTEPWrapperMethod`,
which is the object making the call and not the object the attribute is read from, so it cannot
rescue the lookup.

The history gate was run on a full clone (`git rev-parse --is-shallow-repository` = false).
`git log -G 'submit_write_weight_scale_to_buffer'` over the whole tree returns exactly one
commit, `fcf8882` ("Add avx-based kimi-k2 support", #1656), which added both methods to
`amx.py` and touched `experts.py`, and did not touch `llamafile.py`. Nothing since removes them.
That reads as never built for the other paths rather than deliberately withheld, which is how
the public comment stated it.

**Not verified:** whether the underlying C++ kernels behind the Llamafile and AMX backends can
implement this write-buffer task at all, so no claim is made about which repair is correct. The
crash was not observed on this machine.

Verified 2026-07-31 at `a8062bf` (submodule `1e098a77`). Reported, not fixed upstream at the
time of writing.
