# Prompt-cache token counts dropped from streamed usage metadata

- **Repo:** run-llama/llama_index
- **Surface:** `llms-anthropic` streaming usage aggregation
- **Class:** streaming & usage accounting
- **Fix:** [PR #22311](https://github.com/run-llama/llama_index/pull/22311) (open since
  2026-07-11 and still unreviewed as of 2026-08-04; the defect is unfixed on `main`)

## Root cause

The stream handlers built `usage_metadata` from `input_tokens` and
`output_tokens` only, discarding `cache_creation_input_tokens` and
`cache_read_input_tokens` that the Anthropic SDK returns, in an adapter that
itself supports prompt caching. Cost accounting built on this metadata
silently under-reports.

Anthropic reports the cache counts on `message_start`, and the `message_delta`
handler rebuilds `usage_metadata` from scratch rather than merging into it, so
the final chunk discards them even on the one event that carried them. The
counts are gone at the source: no downstream consumer can recover them, because
they were never passed on.

## Invariant violated

Reported usage must reflect every token class the API returns. If the adapter
enables a billing-relevant feature (prompt caching), its accounting must
account for it.

## Trigger

Any streamed call with prompt caching enabled; the cache token classes vanish
from the reported usage.

## Repro

Mocked SDK event stream (real SDK classes, since the handlers use isinstance
checks): cache fields absent on main, present with the fix. Sibling
integration `llms-bedrock-converse` already captured these fields: the
maintainer-intent precedent.

## Note

Found and fixed by my collaborator [@AmirF194](https://github.com/AmirF194);
included because it completes the streaming-accounting class.
