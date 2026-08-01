# A rescue path gated on length replaces a correct structured extraction with a cruder longer one, and the stage that drops the structure is not the stage that gets blamed

- **Repo:** adbar/trafilatura
- **Surface:** `trafilatura/core.py:224` (`compare_extraction`), `trafilatura/core.py:234-235`
  (the baseline rescue) and `trafilatura/baseline.py:198-201`
  (`trim(elem.text_content())` on the `<article>` branch)
- **Class:** error handling & success reporting
- **Report:** reproduced publicly on
  [issue #896](https://github.com/adbar/trafilatura/issues/896#issuecomment-5151453814)
  (open, filed by another user). No PR from us: the repair is a cascade-ordering decision
  across three stages, and commit `5ca0d20` (PR #646) shows the maintainer chose the
  current tradeoff deliberately, so which stage should carry the guard is his call to make
  before anyone writes it.

## Root cause

Extraction runs as a cascade. Each stage may replace the previous stage's output, and the
test for replacing it is a character count. In Markdown output on a short document, two of
the three stages substitute, and the result keeps neither the heading nor the word
boundaries.

Instrumenting the cascade on a short notice page, against `min_extracted_size` of 250:

```
_extract           -> 244 chars, heading present (<head rend="h1">)
compare_extraction -> 231 chars, heading ALREADY GONE
baseline rescue    -> 243 chars, heading gone, "TitleThis" fused
```

The reporter attributes the whole loss to the rescue at `core.py:234-235`, which is the
only stage visible from the output. That is the last substitution, not the first. The
heading is dropped one stage earlier, by `compare_extraction`, which weighs the main
extractor's result against an external extractor's and returns the one it scores higher.
So a guard placed where the damage appears preserves the 231-character headingless body
instead, and the report is not fixed.

The rescue then fires because 231 is under 250, and `baseline()` on the `<article>` branch
builds its text with `trim(elem.text_content())`. `text_content()` concatenates the text of
every descendant with no separator, so the heading element and the paragraph that follows it
become one run and `Notice Title` joins `This municipal notice` as `TitleThis`. The
structural boundary was the only thing saying those were separate words, and the primitive
that flattens the tree is the primitive that deletes it.

The sanctioned escape hatch does not reach this. `focus != "precision"` guards the rescue, so
`favor_precision=True` skips it: measured, that removes the word fusion and the heading is
still absent, because the heading died at stage 2 and stage 2 runs regardless. `fast=True`
goes the other way, skipping stage 2 so the rescue overwrites the 244-character structured
body directly, and the output is the same fused blob.

## Invariant violated

**A fallback selected by a scalar threshold can only preserve what that scalar measures.**
The rescue asks "is this output too short?" and repairs it by substituting something longer.
"Too short" and "wrong" are different questions, and length is orthogonal to structure: a
correctly parsed 244-character document with a heading scores worse than a 243-character
fused run of the same words. Whenever a quality gate is a single number and the value it
guards is structured, the gate is free to trade away the structure to move the number, and it
will do so silently because the number improved. The tell is a threshold compared against
`len()` of something that is not a string of independent characters.

**A length-based guard cannot rescue this even in principle, and that is worth checking
before proposing one.** The obvious repair is "skip the rescue when the current extraction
already looks good", with `good` meaning long enough. Both candidate outputs here are under
the threshold, 244 and 231 against 250, so every length-based predicate returns the same
answer for the structured body and the fused one. A guard has to test the property that
matters, which is whether the formatted output still carries its block elements.

**In a cascade where several stages may replace a value, the stage where the damage is
visible is almost never the stage where it happened.** The final output is the only artifact
a reporter has, so the blame lands on the last writer by default, and that attribution is
worth exactly nothing until each stage is instrumented separately. One trace of the
intermediate values reorders the whole diagnosis here: it moves the fix site from stage 3 to
stage 2, invalidates the reporter's proposed guard, and shows that the documented workaround
addresses only the half of the symptom that stage 3 caused. Print what each stage returns
before arguing about which one is wrong.

**`text_content()` and its equivalents are lossy in a way that has no error path.** Flattening
a tree to a string discards the fact that block elements imply a boundary, so words that were
never adjacent become adjacent. Nothing raises, nothing logs, and the output stays plausible
prose, which is why it survives review and reaches a user as a corrupted word rather than a
crash.

## Trigger

Any document whose extracted body lands under `MIN_EXTRACTED_SIZE` (250 characters) in the
default balanced mode, which is the shipped path. Short pages reach it easily: notices,
stubs, single-section pages. The reporter frames it as a 2.2.0 regression and the history
agrees on the fusion half: `git log -G` on the `text_content` call returns one commit,
`18a7b42` (PR #881, "revamped text recovery and extraction sequence"), which is in the 2.2.0
tag and no earlier one. Measured against 2.1.0, the same input keeps its heading, and the
reason is incidental rather than reassuring: 2.1.0 emitted a duplicated paragraph, which
carried the length past 250 and kept the rescue from firing at all.

## Repro

Clean `python:3.12-slim` container at `c1bc9531`, which is still `master`. A short HTML
notice with an `<h1>` and one paragraph is run through `trafilatura.extract` with
`output_format="markdown"`, then through the same cascade with each stage's return value
printed, so the 244 / 231 / 243 figures above come from the library's own functions rather
than from reasoning about them. The same input is run under `fast=True` and
`favor_precision=True` to separate which stage each flag actually skips, and against the
2.1.0 release wheel for the regression window.

The history was read on a full clone (`git rev-parse --is-shallow-repository` returns
`false`), with both probe strings escaped for regex metacharacters and cross-checked with
`grep -cF` so that a zero-hit result would have been read as a broken probe rather than as an
absence of history. That is what surfaced `5ca0d20` (PR #646, 2024-07-18), which introduced
the `focus != "precision"` condition on the rescue and establishes that the rescue firing in
balanced mode is intended behaviour rather than an oversight.

**Not verified:** one document shape was measured, not a corpus, so no claim is made about
how often short pages hit this in the wild. The word fusion was confirmed only on the
`<article>` branch of `baseline()`; the paragraph branch was not exercised. No fix was
written or tested.

Verified 2026-08-01 at `c1bc9531`, `master` at the time of writing. Reported upstream, open,
with no maintainer reply and no PR from anyone claiming the fault.
