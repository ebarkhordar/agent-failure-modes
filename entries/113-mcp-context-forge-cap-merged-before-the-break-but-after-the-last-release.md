# A dependency cap merged one day before the breaking release still protects nobody, and the import that fails names a package other than the one to constrain

- **Repo:** IBM/mcp-context-forge
- **Surface:** `pyproject.toml:142` at HEAD, against the metadata baked into the published
  `mcp-contextforge-gateway` 1.0.6 on PyPI. The frame that raises is in a transitive
  dependency: `cpex/framework/external/mcp/client.py:24`
- **Class:** release paths & published artifacts
- **Report:** reproduced publicly on
  [issue #6016](https://github.com/IBM/mcp-context-forge/issues/6016#issuecomment-5142844131)
  (open, filed by another user hitting it on the Quick Start). No PR from us: the cap is
  already on `main`, so what remains is a release decision (cut one, or say so on the Quick
  Start page), which is the maintainers' to make and not something a patch can express.

## Root cause

`mcp-contextforge-gateway` 1.0.6 was published on 2026-07-21 declaring `mcp>=1.28.1` with no
upper bound. `mcp` 2.0.0 was published on 2026-07-28 and removed the `McpError` alias, keeping
`MCPError`. A fresh install of the latest release therefore resolves `mcp` 2.0.0 and fails at
import before the process can serve anything:

```
ImportError: cannot import name 'McpError' from 'mcp'. Did you mean: 'MCPError'?
```

The repository is not unaware of this, and it did not react late. Commit `91f8920c2`
(PR #5840, Ahmad Al Tamimi, 2026-07-27 12:51 UTC) changed `pyproject.toml:142` to
`mcp>=1.28.1,<2`, which is roughly a day *ahead* of the upstream 2.0.0 upload. Read `main` and
the bug is fixed, correctly and pre-emptively.

`git tag --contains 91f8920c2` returns nothing. The newest tag is `v1.0.6`, which is the
release that predates the cap, so the cap is in no published artifact. The maintainers won the
race against upstream inside the repository by one day and lost it on the index by ten.

The second half is where triage goes wrong. The traceback's last frame is inside `cpex`, not
inside `mcpgateway`, and `cpex` 0.1.2 also declares `mcp>=1.26.0` uncapped. Reading the error
suggests `cpex` is the project to fix. It is not: `pip` resolves a single `mcp` for the
environment, so the gateway's own `<2` constrains what `cpex` will import, and the gateway is
both the package the user asked for and the one whose release cadence is in scope.

## Invariant violated

**A version cap takes effect when a release carrying it is published, not when it merges, so
the useful unit of "fixed" is the index and never the branch.** This is [entry
106](106-harness-sdk-release-guard-excludes-its-own-first-release.md)'s second invariant
observed under a different obstruction, and the pair is the reason to record it twice. There,
the release job structurally could not run. Here nothing was blocking at all: no CI was red, no
guard refused, no reviewer objected. A correct fix simply sat on `main` while no release
happened to be cut, which is the more common and much quieter of the two shapes. A cap's
correctness is therefore a function of release cadence, and a project that merges dependency
bounds faster than it publishes has a fix latency equal to its release interval no matter how
early anyone reacts.

**In a resolver that installs one version of a package per environment, the constraint that
repairs a break may live at any node of the dependency graph, but the traceback can only ever
name the node that ran the import.** So the package in the error message is close to the least
informative fact available, and the instinct it produces (go read that project) points away
from the fix. The question worth asking on an `ImportError` after a major bump is not "who
imported it" but "who in this graph is allowed to constrain it", and for a single-version
resolver the answer is anyone, which means the top-level package the user actually named is
usually both sufficient and the right owner. Capping `cpex` here would work and would still be
the wrong repair, because it treats a graph-wide property as if it were local to the frame that
happened to raise.

**A dependency cap is the one class of fix that every test in the repository is structurally
unable to check.** CI installs from the source tree with the cap present and passes; resolution
against an index using metadata baked into a built artifact is a different operation that the
suite never performs. That is what lets a project sit in the state above without a single
signal, and it is why "is it fixed on main" and "is it fixed for users" have to be asked as two
separate questions here.

## Trigger

A fresh install of the latest published `mcp-contextforge-gateway` (1.0.6) on or after
2026-07-28, by any resolver without a pre-existing `mcp<2` constraint in the environment. Not
platform specific: the reporter hit it on Windows, the run below is Linux. Not reachable in the
project's own CI, which installs from a tree that carries the cap.

## Repro

Clean `python:3.12-slim`, installing exactly the way the Quick Start does:

```
$ pip install "mcp-contextforge-gateway==1.0.6"
$ python -c "import importlib.metadata as m; print(m.version('mcp'))"
2.0.0
$ python -c "import importlib.metadata as m; print([r for r in m.requires('mcp-contextforge-gateway') if r.startswith('mcp>')])"
['mcp>=1.28.1']
$ python -c "import importlib.metadata as m; print(m.version('cpex'), [r for r in m.requires('cpex') if r.startswith('mcp>')])"
0.1.2 ['mcp>=1.26.0']
$ python -c "import mcp; print('McpError', 'McpError' in dir(mcp), '| MCPError', 'MCPError' in dir(mcp))"
McpError False | MCPError True
$ mcpgateway --host 127.0.0.1 --port 4444
  File ".../cpex/framework/external/mcp/client.py", line 24, in <module>
    from mcp import ClientSession, McpError, StdioServerParameters
ImportError: cannot import name 'McpError' from 'mcp'. Did you mean: 'MCPError'?
```

The same container with the constraint added boots, runs its migrations and serves:

```
$ pip install "mcp-contextforge-gateway==1.0.6" "mcp<2"
$ python -c "import importlib.metadata as m; print(m.version('mcp'))"
1.29.0
$ mcpgateway --host 127.0.0.1 --port 4444
(boots; still serving when the run was cut at 25s)
```

The release claim was measured on a full clone (`git rev-parse --is-shallow-repository` =
false, so `git tag --contains` is reading real history): `91f8920c2` is an ancestor of
`origin/main`, is contained in no tag, and `pyproject.toml:142` reads `mcp>=1.28.1,<2` at HEAD.
PyPI upload timestamps were read from the JSON API (`mcp-contextforge-gateway` 1.0.6 at
2026-07-21T17:26Z, `mcp` 2.0.0 at 2026-07-28T13:45Z).

**Not verified:** no other resolver was exercised (`pip` only, no `uv` or Poetry), and no claim
is made about which other published packages in this graph break under `mcp` 2.0.0. Whether a
release is imminent is not something the repository states, so this entry says only that none
carrying the cap exists as of the date below.

Verified 2026-07-31, gateway `main` at `91f8920c2`'s descendant tip. Reported, not released
upstream at the time of writing.
