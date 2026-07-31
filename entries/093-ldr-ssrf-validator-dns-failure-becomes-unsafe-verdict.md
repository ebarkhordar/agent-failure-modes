# An SSRF validator returns "unsafe" when DNS fails, so losing the resolver reads as a blocked address

- **Repo:** LearningCircuit/local-deep-research
- **Surface:** `src/local_deep_research/security/ssrf_validator.py`, `validate_url`
  (`:326-328`), reached from `is_safe_custom_llm_endpoint`; against the call at
  `web/routes/research_routes.py:657`, which was ungated when this was reported
- **Class:** error handling & success reporting
- **Report:** reproduced publicly on
  [issue #5220](https://github.com/LearningCircuit/local-deep-research/issues/5220#issuecomment-5080275355)
  (closed as completed 2026-07-30). Filed as triage rather than a patch: the correct arm
  (fail open on resolution failure, gate the check on the provider, or both) is a security
  policy decision the maintainer owns, not a defect with one obvious repair.
- **Fix:** partial.
  [PR #5255](https://github.com/LearningCircuit/local-deep-research/pull/5255), by another
  contributor, merged 2026-07-30, took the second arm only; the conflation in the
  validator is unchanged. See "Resolution" below.

## Root cause

`validate_url` resolves the hostname so it can reject addresses that live inside the
private ranges. Resolution and policy share one exception handler:

```python
except socket.gaierror:
    logger.warning(f"Failed to resolve hostname {hostname}")
    return False
```

`False` is the same value the function returns for a hostname that resolves to
`127.0.0.1`. The caller receives one boolean and cannot distinguish "this address is
forbidden" from "I could not find out", so a container that has lost outbound DNS
rejects every hostname-based endpoint with `Invalid custom endpoint URL`.

Two independent facts turned that conflation, at the commit reported, from a corner case
into a total outage of the feature. First, the endpoint was validated on every research
run regardless of provider: `custom_endpoint` is added to the submitted form at
`web/static/js/components/research.js:2670` unconditionally, and
`research_routes.py:657` validates it with no guard, while the required-field check
eighteen lines above at `:639` *is* gated on `model_provider == "openai_endpoint"`.
Second, the field is populated from the `llm.openai_endpoint.url` setting whose shipped
default is `https://openrouter.ai/api/v1` (`defaults/default_settings.json:594`). So a
user running `LDR_LLM_PROVIDER=lmstudio` against a LAN address submits, and has
validated, a public hostname the run will never contact; `research_routes.py` contains
no reference to lmstudio at all.

Put together: a default value nobody chose, validated on a path that does not use it,
by a function that reports a network failure as a policy denial.

## Invariant violated

A predicate that answers a security question must not answer an availability question
with the same value. "Forbidden" and "unknown" are different verdicts, and collapsing
them into one boolean moves the decision from the caller (which knows whether it can
afford to fail open) into the resolver (which knows nothing about the caller).

Fail-closed is the right default for an SSRF check, and that is precisely why this is
easy to ship: the code reads as the cautious choice, and the cautious choice is correct
for the case the author had in mind, an attacker-supplied name that does not resolve.
It is wrong for the case the author did not have in mind, the resolver itself being
gone, because then the predicate denies everything, uniformly, with a message about the
address. The user reads a claim about their input and starts editing their input, which
is the one thing that cannot help.

The generalizable rule: when a guard needs an external lookup to reach its verdict,
the lookup's failure is a third outcome, and something must decide what it means. Model
it as three states (allowed, denied, indeterminate) and make the indeterminate arm an
explicit policy at the call site, or at minimum report it with a distinct message, so
the operator is told "cannot resolve this host" rather than "this host is invalid". A
diagnostic that names the wrong cause is worse than a generic one, because it is
actionable in the wrong direction.

Two nearby smells travel with this shape and both are visible here. A validator applied
to a value the code path will not use extends the blast radius of every one of the
validator's own bugs into paths that had no reason to run it. And a non-empty default
for an optional external endpoint means the validator has an input to reject even when
the user has configured nothing, so the feature is never in the "unset" state where the
check is skipped (an empty endpoint is treated as safe, `utilities/url_utils.py:198-199`,
so clearing the field is the workaround).

## Trigger

Any deployment where the container cannot reach a DNS resolver. At the reported commit
this held with any LLM provider selected, and every research run failed with `Invalid
custom endpoint URL`, naming a URL the user may never have configured and the run would
never contact; since PR #5255 it is confined to `openai_endpoint`, where the URL is at
least one the user chose. An IP literal endpoint is unaffected in both cases, since no
resolution is attempted.

## Repro

Clean `python:3.11-slim` with the package's runtime dependencies, HEAD `abe62643`, the
same image and commit for both runs so that only the container's resolver differs:

```
docker run --rm ...                     # working DNS
  https://openrouter.ai/api/v1    resolves=104.18.3.115                       is_safe=True
  http://192.168.178.52:1234/v1   resolves=192.168.178.52                     is_safe=True

docker run --rm --dns 203.0.113.1 ...   # no reachable resolver
  https://openrouter.ai/api/v1    resolves=gaierror (name resolution failure) is_safe=False
  http://192.168.178.52:1234/v1   resolves=192.168.178.52                     is_safe=True
```

The IP literal passing in both runs is what isolates the mechanism to resolution rather
than to address policy: the same validator, the same commit, and the only input that
changes verdict is the one that needs DNS.

Verified 2026-07-25 at HEAD `abe62643`. Re-checked 2026-07-26 at HEAD `ffba6176`:
`validate_url` still returns `False` from the `socket.gaierror` handler at `:326-328`.

## Resolution

PR #5255 merged 2026-07-30 and the owner closed #5220 as completed the same day. Its
source change is entirely in `web/routes/research_routes.py`: `custom_endpoint` is read
only when `model_provider == "openai_endpoint"`, and the validation call at `:657`
carries the same guard. That removes the amplifier described above, so the outage no
longer reaches users on lmstudio, ollama or any other provider, and it was the right
first move because it is the arm that needed no security judgement.

The predicate itself was not touched. Re-checked at `main` `93433917`:
`ssrf_validator.py:326-328` still returns `False` from the `socket.gaierror` handler, so
for a user who has actually selected `openai_endpoint`, a container that loses its
resolver still reports `Invalid custom endpoint URL` about an address that is not
invalid. The population is smaller; the diagnostic still names the wrong cause.

That split is the part worth keeping. Two defects reached users through one symptom, and
only one of them was a defect in the component the symptom named. Fixing the reachability
retires the report, closes the issue, and satisfies every observer, while the confusion
between "denied" and "could not determine" stays in the security layer where it will be
reached again by the next caller. When a report is resolved by narrowing who can reach the
faulty code, the fault has not been adjudicated, and nothing in the closed issue records
that it is still there.
