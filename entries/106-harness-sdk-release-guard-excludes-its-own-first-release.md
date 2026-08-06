# The fix is in the repository and cannot reach users, because the release job refuses to run without a baseline tag that this component did not have

- **Repo:** strands-agents/harness-sdk
- **Surface:** `.github/workflows/release-mcp.yml`, the `scan-commits` job (`:81-86`), against
  the published `strands-agents-mcp-server` 0.2.7 on PyPI and the cap already present at
  `strands-mcp/pyproject.toml:27`
- **Class:** release paths & published artifacts
- **Report:** reproduced publicly on
  [issue #3533](https://github.com/strands-agents/harness-sdk/issues/3533#issuecomment-5113963614),
  closed as completed 2026-08-06 by a maintainer after 0.2.8 reached PyPI. No PR from us: the
  two candidate repairs are pushing a baseline `mcp/v0.2.7` tag by hand and carving a
  first-release case out of the guard, and which one is correct depends on release policy we
  cannot read from the repository. Upstream took the first. See **Resolved 2026-08-06** below,
  which also corrects one word of the headline.

## Root cause

The published `strands-agents-mcp-server` 0.2.7 declares `mcp>=1.1.3` with no upper bound. `mcp`
2.0.0 removed `mcp.server.fastmcp`, so resolving the package fresh installs a dependency it
cannot import:

```
ModuleNotFoundError: No module named 'mcp.server.fastmcp'
  strands_mcp_server/server.py, line 4, in <module>
```

The repository is not unaware of this. `strands-mcp/pyproject.toml:27` carries
`mcp>=1.1.3,<2.0.0` at HEAD, and `:67` carries the same bound for the static-analysis
environment. Read the source, and the bug is fixed.

The defect is that the cap cannot ship. The release workflow's first job resolves the previous
release from a component-scoped tag namespace and refuses to continue without one:

```yaml
TAG_PREFIX="mcp/v"                                                          # :81
PREV_TAG=$(git tag --list "${TAG_PREFIX}*" --sort=-v:refname | head -n1)    # :82
if [ -z "$PREV_TAG" ]; then                                                 # :83
  echo "::error::No prior tag matching ${TAG_PREFIX}* ... refusing to release without a baseline."
  exit 1
fi
```

The repository carries 164 tags, under `python/` (69), `typescript/` (35), `python-wasm/` (1) and
bare `v*` (59). Under `mcp/` there are none, which is the condition `:83` exits on. Every later
job in that workflow lists `scan-commits` in `needs` (`:132`, `:142`, `:152`, `:245`, `:317`,
`:342`), so a `workflow_dispatch` stops in the first job and nothing is built, signed or
uploaded.

The blast radius is one package. Reading `requires_dist` from PyPI for the rest of the family:
`strands-agents` 1.50.2 already ships `mcp<2.0.0,>=1.23.0`, and `strands-agents-tools` 0.8.5 and
`strands-agents-builder` 0.1.10 declare no `mcp` dependency at all. Only 0.2.7 is unbounded.

## Invariant violated

**A guard that refuses to proceed without a previous success excludes its own first run.** The
rule at `:83` is correct for every release after the first: without a baseline there is no
commit range to scan and no version to compare against, so refusing is safer than guessing. What
the rule has no case for is the state the component actually starts in. A monotonic-progress
check needs a defined answer at n=0, otherwise the mechanism that protects every subsequent
release is precisely what prevents the first one, and the failure is a fixed point: the guard
demands a tag that only a successful run of the guarded workflow would create.

This shape hides well in a monorepo. The workflow was written and exercised in namespaces that
were already populated, where the guard has fired correctly hundreds of times across 164 tags.
A newly carved-out component enters at n=0 by construction, and it is the only place in the repo
where the empty case is reachable, so the guard's test surface and its defect surface do not
overlap at all. Coverage is not the property that would have caught this. Asking what each guard
does on an empty input is.

The second invariant is the one that makes the first one dangerous rather than merely annoying.
**The artifact users install is the published one, and a repository's own tests exercise the
source tree, so a dependency constraint is unverified by everything in CI until a release is
cut.** A cap added to `pyproject.toml` is a claim about resolution behaviour, and resolution
happens at install time, from an index, against metadata baked into a built artifact. Every test
that passes on the branch passes with the pinned development environment, which is why the
change looks landed. Worse, the presence of the fix in the source is actively misleading during
triage: a maintainer or a reporter who checks `pyproject.toml` sees `<2.0.0` and concludes the
issue is closed pending a release, when what is actually true is that no release can be produced.
The gap between HEAD and the index is only visible from the index.

## Trigger

Any fresh install of the published `strands-agents-mcp-server` after `mcp` 2.0.0 was released:
`uvx strands-agents-mcp-server`, or a `pip install` into an environment with no existing `mcp`
pin. Pre-existing installs with a resolved `mcp` 1.x are unaffected, so the break appears only
for new users and new environments, which is the population least likely to have somewhere to
report it.

## Repro

Clean `python:3.12-slim`:

```
$ uvx strands-agents-mcp-server
Traceback (most recent call last):
  File ".../bin/strands-agents-mcp-server", line 6, in <module>
    from strands_mcp_server.server import main
  File ".../site-packages/strands_mcp_server/server.py", line 4, in <module>
    from mcp.server.fastmcp import FastMCP
ModuleNotFoundError: No module named 'mcp.server.fastmcp'
```

Resolved in that container: `strands-agents-mcp-server` 0.2.7 with `mcp` 2.0.0.
`uvx --with "mcp<2" strands-agents-mcp-server` installs 30 packages on the same image and starts
without the traceback, which confirms the version boundary is the whole cause and not a second
incompatibility hiding behind it. The family metadata came from the PyPI JSON API, the tag
namespaces from an unpaginated `git ls-remote --tags`, and the workflow and `pyproject.toml`
citations were read at HEAD `f2482f41`.

Not checked: whether a `workflow_dispatch` has in fact been attempted and failed at `:83`. The
guard's behaviour on an empty tag list is read from the workflow source and from the measured
absence of any `mcp/` tag, not from an observed run.

Verified 2026-07-29. Reported, not fixed upstream at the time of writing.

## Resolved 2026-08-06

`strands-agents-mcp-server` 0.2.8 was uploaded to PyPI on 2026-07-30T21:30:16Z and its
`requires_dist` carries `mcp<2.0.0,>=1.1.3`, so the cap that sat in `pyproject.toml` is now in
the published metadata and `uvx strands-agents-mcp-server` resolves `mcp` 1.x. A maintainer
closed #3533 as completed on 2026-08-06 after another user re-ran the install and reported it
clean.

Of the two candidate repairs named above, upstream took the first. `.github/workflows/release-mcp.yml`
has exactly one commit in its history (`90d63bab`, 2026-07-24), so the guard was never touched:
`if [ -z "$PREV_TAG" ]` is still at `:83` and still exits 1 on an empty tag list. What changed is
the input. The repository now answers `git ls-remote --tags` with 175 refs against the 164
measured on 2026-07-29, and ten of them are under `mcp/`: `v0.1.0`, `v0.2.0` through `v0.2.8`.
Nine predate the new release and point at their historical commits (`mcp/v0.1.0` at `b3a5bd68`,
committed 2025-05-16; `mcp/v0.2.7` at `0b7ea7cd`, committed 2026-03-10), which is what a
backfill looks like and not what a release produces. It is a reconstruction rather than a
record: the namespace and PyPI's twelve uploads do not line up either way, since `mcp/v0.2.1`
names a version never published and 0.0.1, 0.1.1 and 0.1.2 were published and got no tag. Only
the highest tag has to be right for `:82` to work, which is presumably why the rest was cheap. The GitHub API lists two runs of that workflow, both `workflow_dispatch` and both
successful, at 2026-07-30T17:59Z and 18:25Z; their run numbers are 3 and 4, so two earlier runs
are missing from the list and this entry does not know what they were.

The dates order the events and nothing here establishes that the report caused the repair.

**The defect the entry is about is unfixed; only this instance of it is.** The guard still has
no case for an empty namespace, so the next component carved into this monorepo starts at n=0
against the same `exit 1`, and the repair that worked here has to be remembered and repeated by
hand. That is the ordinary shape of fixing an n=0 bug by supplying the missing n: the fixed
point is broken for one namespace, not removed.

One word of the original write-up was wrong, and the correction is the same rule this corpus
applies to a PR. The headline and the Root cause section said the component had **never** had a
tag under `mcp/`. The measurement behind that was a single unpaginated `git ls-remote --tags` on
2026-07-29, which is an instrument that reads the refs a repository has now. It cannot see a
deleted tag, and a backfilled tag is indistinguishable from an always-present one in it, so
"there are none today" was supported and "there never have been" was not. Both now read
"did not have". The `:83` diagnosis never depended on the stronger claim: an empty tag list at
dispatch time is the whole trigger.
