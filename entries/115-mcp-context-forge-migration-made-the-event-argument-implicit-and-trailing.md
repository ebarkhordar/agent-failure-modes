# A migration that made the event argument implicit and trailing broke exactly the four handlers whose old markup had passed it first

- **Repo:** IBM/mcp-context-forge
- **Surface:** the delegated dispatcher in `mcpgateway/admin_ui/eventDelegation.js:106`
  (`return fn(...args, event)`), with the empty-args value push at `:166-168` (input) and
  `:184-192` (change), against the `data-action-input` / `data-action-change` bindings
  declared in `mcpgateway/templates/admin.html`
- **Class:** indexing, ordering & counting contracts
- **Report:** reproduced publicly on
  [issue #6001](https://github.com/IBM/mcp-context-forge/issues/6001#issuecomment-5150193235)
  (open, filed by another user who hit it on the chat textarea). No PR from us: two open PRs
  already patch this file at the leaf, and the repo's CONTRIBUTING asks for triage before
  implementation.

## Root cause

Content Security Policy work removed the inline `on*` attributes from the admin templates and
replaced them with declarative bindings that name a handler. The dispatcher then supplies the
arguments itself:

```js
return fn(...args, event);
```

`args` comes from the element's `data-arg*` attributes, and for an input or change binding that
declares none, the dispatcher pushes the element's current value. A handler bound with no
`data-arg0` is therefore called as `fn(value, event)`. Driving the real
`initializeEventDelegation()` against a DOM node carrying those attributes, the argv is:

```
input  handler argv types: ["String","Event"]
change handler argv types: ["String","Event"]
```

so a handler written `function h(event)` binds `event` to the value string.

The markup this replaced was written once per call site, so it could express both shapes:

```
-  onchange="Admin.handlePerformanceAggregationChange(event)"
+  data-action-change="handlePerformanceAggregationChange"
-  onchange="Admin.validateCACertFiles(event)"
+  data-action-change="validateCACertFiles"
```

The new attribute has one arity for everybody. The two remaining cases were inline
`this.style.height = ...` and `oninput="this.setCustomValidity('')"` bodies that the same
migration rewrote into `event.target` helpers, so they acquired an event parameter at the exact
commit that stopped passing one first.

Of the 20 distinct actions bound with no `data-arg0` (22 binding sites), four take the event
first:

| binding | handler | observed with the shifted args |
|---|---|---|
| `admin.html:1738` (chat textarea) | `autoResizeTextarea`, `eventDelegation.js:353` | `TypeError: Cannot read properties of undefined (reading 'style')` |
| `admin.html:6305` (`expires_in_days`) | `clearCustomValidity`, `eventDelegation.js:345` | `TypeError: ... (reading 'setCustomValidity')` |
| `admin.html:823` (`performance-aggregation-select`) | `handlePerformanceAggregationChange`, `logging.js:81` | no throw: returns `undefined`, never reaches `showPerformanceMetrics` |
| `admin.html:5859` (`upload-ca-certificate`) | `validateCACertFiles`, `caCertificate.js:12` | `TypeError: ... (reading 'files')` |

The other 16 take the value or no parameter, so today's dispatch is correct for them.

## Invariant violated

**A migration that replaces a per-call-site convention with one implicit rule can only encode
the majority case, and it silently converts every site that was an exception into a wrong one
without editing that site's code.** The property that broke here is not reachability: all 22
bindings still resolve, still fire, and still call their handler, so the check a migration
naturally gets tested against passes everywhere. What broke is arity and order, which no
individual diff hunk displays, because each hunk shows one attribute becoming one attribute.

The second half is why the audit is hard afterwards. **The evidence needed to verify the
migration was the thing the migration deleted.** Which sites passed `event` was recorded only
in the `on*` attributes, so after the commit there is no artifact anywhere in the tree that
distinguishes a handler that wants the value from one that wants the event. It is recoverable
only from history, and only if you know to look. A conversion that destroys its own input is a
conversion whose correctness has to be established before it lands, never after.

**Loudness is uncorrelated with severity here, and the bug report is a sample biased toward the
noisy end.** The four failures span the entire range: two throw a `TypeError` into the console,
one throws on its first line so that no client-side size or extension check on a CA certificate
upload runs at all, and one reads `event?.target?.value` through an optional chain, returns
`undefined`, and produces no error whatsoever. The reported case is a textarea that does not
resize. The unreported ones include a security-adjacent validator that never executes and a
selector control that quietly does nothing. So the set of affected sites is a question about
program structure, answered by parsing every binding and reading the first parameter of each
target handler, and it is not a question a user's report can answer even in principle.

**A defect that presents at N call sites attracts N local patches, each individually correct and
none of them the fix.** Both open PRs here are exactly that. #5045 skips the value push when
`target.type === 'file'`, which covers `validateCACertFiles` and nothing else. #4801 rewrites
`autoResizeTextarea` to `const el = event?.target; if (!el?.style) return;`, which removes the
console error, and returns before the height is ever set, so the textarea still does not resize.
That second one is worth dwelling on: it converts a loud failure into a silent one, which is the
direction that ends the reporting and therefore ends the chance of a real fix. An opt-in marker
at the binding, the way the repo's own `handleDelegatedSubmit` already calls
`executeAction(action, args, event, true)`, is the shape that covers all four in one place and
gives the next migrated handler somewhere to declare itself.

## Trigger

Interacting with any of the four bindings in the default admin UI. No configuration, no
credentials, no special build: typing in the LLM Chat textarea, editing the token expiry field,
changing the performance aggregation selector, or selecting a CA certificate file. Handlers that
declare their own `data-arg*` are on a different path and were not examined.

## Repro

Clean `node:22` container, `npm ci`, then a throwaway spec on the repository's own vitest and
jsdom harness at `main` `a48826ba0`, driving the real `initializeEventDelegation()` rather than a
reimplementation of it. The spec attaches a node with `data-action-input` / `data-action-change`
and no `data-arg*`, dispatches the event, and prints the constructor names of the handler's
arguments; that is the `["String","Event"]` output above. Each of the four handlers was then
invoked through the same dispatcher to obtain the right-hand column of the table.

The affected set was established by parsing the templates for every `data-action-input` and
`data-action-change` binding and reading the first parameter of each named handler, not by
searching for the symptom. That is what produced 20 distinct actions across 22 sites and the
four-of-twenty split.

The origin was read on an unshallowed clone:
`git log -G 'return fn\(\.\.\.args, event\)' --follow` on `eventDelegation.js` returns exactly
one commit, `c9df29c88` (PR #4673, the CSP inline-handler migration), whose diff contains the
attribute rewrites quoted above. That makes the arg order migration fallout rather than a
deliberate design call, though the people who ran #4673 would know better than an outside reader.

**Not verified:** the bindings that declare their own `data-arg*` were not audited, so no claim
is made about them. Whether the server re-validates CA certificate uploads was not checked, so
the impact of `validateCACertFiles` never running is stated as client-side only. Nothing was
committed for the probe; the spec was deleted after the run.

Verified 2026-08-01 at `main` `a48826ba0b`, which is still the tip as of this entry. Reported,
not fixed upstream at the time of writing.
