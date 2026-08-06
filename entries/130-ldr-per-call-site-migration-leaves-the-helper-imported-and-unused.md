# A migration done one call site at a time leaves files that import the coercion helper and read the raw field eleven times beside it

- **Repo:** LearningCircuit/local-deep-research
- **Surface:** `src/local_deep_research/advanced_search_system/strategies/topic_organization_strategy.py`
  (helper imported at `:16`, called at `:761`, seven raw reads elsewhere),
  `news/core/news_analyzer.py` (imported at `:13`, called at `:153`, three raw reads) and
  `web_search_engines/engines/search_engine_github.py:168` (one raw read), against
  `utilities/json_utils.get_llm_response_text`
- **Class:** message-conversion boundaries
- **Fix:** [PR #5422](https://github.com/LearningCircuit/local-deep-research/pull/5422) (open, in
  review; refs the umbrella issue
  [#4615](https://github.com/LearningCircuit/local-deep-research/issues/4615))

## Root cause

LangChain hands back a message whose `content` is a `str` for most providers and a list of
typed blocks for Anthropic extended thinking and tool use. `get_llm_response_text` exists to
absorb that difference: it flattens the block list to text and strips `<think>` tags. PR #2220
introduced it and converted one call site in each of these three files, leaving the rest. The
result is a file that imports the helper, uses it once, and reads `response.content` raw eleven
more times.

The same wrong value then fails three different ways, and which way depends entirely on what the
consumer does next:

`topic_organization_strategy.py` appends the value to `generated_texts`, so `"\n".join(...)`
raises `TypeError: sequence item 0: expected str instance, list found`. Five sibling sites
stringify the block list instead, so `int(response_text)` fails and lead re-selection keeps the
original source without saying anything. `_filter_topics_by_relevance` tests for `"yes"` anywhere
in that same repr, which includes the thinking block, so a topic the model rejected in its answer
is kept because the reasoning it discarded contained the word.

`news_analyzer.py` calls `.strip()` on the value in `generate_big_picture`, `generate_watch_for`
and `generate_patterns`. On a list that is an `AttributeError`, raised inside a broad `except`,
so each returns empty and the news card loses those sections with nothing in the log.

`search_engine_github.py` runs `str()` over the list, so the query sent to GitHub's search API
becomes `[{'type': 'text', 'text': 'agents language:python stars:>100'}]`. The API accepts it and
returns whatever that matches.

## Invariant violated

**An import is a claim about what a file knows, and a per-call-site migration makes it false.**
The helper's import at the top of `topic_organization_strategy.py` is the strongest evidence
available that this file's author knew about list content and knew where the fix lives. It is
also what makes the seven remaining raw reads unambiguously an oversight rather than a decision,
which is what made this a safe change to propose. The same property is what hides it: a reviewer
asking "does this file handle block lists" finds the import and one correct call, and both
answers are yes.

**A cast added to satisfy a type checker converts a crash into a wrong value, and that is a
downgrade.** The `str()` wrapper in `search_engine_github.py` arrived in #2910 for mypy. mypy
asked for a `str` and got one. What it cannot ask is whether the string means anything, and the
`AttributeError` that would have surfaced this call site was the only thing making it visible.
Silencing a type error by widening the value rather than fixing the type is how a loud failure
becomes a query that returns plausible results forever.

**Any `in` test over a stringified structure silently widens its search to the parts of the
structure nobody meant to read.** `"yes" in str(blocks)` searches the thinking block, the field
names, and the block types, none of which is the model's answer. The repr of structured data is
still a string, so the test type-checks, runs, and reports on a haystack that grew without
anybody choosing it. The failure direction is the dangerous one: extra text can only add matches,
so a relevance filter built this way fails toward keeping things.

**Severity is set by the consumer, not by the defect, so counting call sites tells you nothing
about how bad it is.** One value, eleven sites, three outcomes: a traceback, an empty section
swallowed by an `except`, and a wrong request to a third party. The loudest one is the only one
anybody would have reported, and it is the least costly of the three.

## Trigger

Any Anthropic model with extended thinking or tool use enabled, reaching any of the eleven sites.
No configuration in this repository turns it on or off; the shape of `content` is the provider's
choice, so the same code path is correct with one model and broken with another.

## Repro

In a container at head `7e3b664f`, a fake model returning Anthropic-style block lists was driven
through all eleven sites, and each of the three failures above was observed before being fixed,
including the `_filter_topics_by_relevance` case where a topic whose text block said "no" was
kept because the thinking block contained "yes". Ten regression tests, one per behaviour, fail on
`main` and pass on the branch. `tests/advanced_search_system/`, `tests/web_search_engines/` and
`tests/news/` produce an identical set of 163 pre-existing failures and errors before and after
the change (optional dependencies absent from the container), the only difference being the ten
added passes.

The scope came from an `ast` walk of `src/local_deep_research` rather than a text search: 39 raw
`.content` reads across 21 files, of which 3 files import the helper and call it exactly once,
and 18 files (28 reads) never reference it at all. Those 28 are deliberately not in this PR: in a
file that never imported the helper, a raw read is not visibly an oversight, and the sweep is a
different and much larger change.

**Not verified:** no live provider call. Every block list here is constructed, so the claim is
that this shape breaks these sites, not that a particular Anthropic release emits exactly this
shape today.

Verified 2026-08-06 at `7e3b664f`.
