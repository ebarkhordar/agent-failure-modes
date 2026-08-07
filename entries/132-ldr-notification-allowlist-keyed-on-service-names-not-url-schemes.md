# A scheme allowlist populated with service names rejects every URL that works and admits five that no plugin will ever claim

- **Repo:** LearningCircuit/local-deep-research
- **Surface:** `src/local_deep_research/security/notification_validator.py`,
  `NotificationURLValidator.ALLOWED_SCHEMES` (`:47-66`) and the membership test that reads it
  (`:406`), reached from `NotificationService.test_service`
  (`src/local_deep_research/notifications/service.py:258`, gate at `:298`), against
  apprise 1.12.0's plugin scheme registry
- **Class:** configuration wiring & documented contracts
- **Report:** [issue #5399](https://github.com/LearningCircuit/local-deep-research/issues/5399)
  (open), reproduced in
  [our comment](https://github.com/LearningCircuit/local-deep-research/issues/5399#issuecomment-5201616653)
  on 2026-08-06. No PR from us: the repair is a policy choice between two shapes (correct the
  literal list, or derive it from apprise's own registry at import time) and the maintainer
  owns that call, so the comment reproduces the defect and asks which shape he wants rather
  than asserting one.

## Root cause

`ALLOWED_SCHEMES` is a frozen tuple of strings compared against the scheme of a user-supplied
notification URL:

```python
ALLOWED_SCHEMES = (
    "http", "https", "mailto", "discord", "slack",
    "telegram",     # Telegram bot API
    "gotify",
    "pushover",     # Pushover notifications
    "ntfy", "ntfys", "signal", "matrix",
    "mattermost",   # Mattermost webhooks
    "rocketchat",   # Rocket.Chat webhooks
    "teams",        # Microsoft Teams
    "json", "xml", "form",
)
```

```python
if scheme not in NotificationURLValidator.ALLOWED_SCHEMES:
    ...  # "Unsupported protocol: {scheme}"
```

The URLs this list gates are apprise URLs, and apprise dispatches on a plugin's `protocol` /
`secure_protocol`, not on the service's name. For five of the eighteen entries the two are
different strings. Telegram's scheme is `tgram`, Pushover's is `pover`, Mattermost's is
`mmost`, Rocket.Chat's is `rocket` / `rockets`. Each of those four names is the filename of
the plugin module rather than anything apprise will parse, so the list reads like it was
assembled by looking at `apprise/plugins/` instead of at the URLs users type.

The fifth is a different error wearing the same shape. `teams` is not a stale spelling of an
apprise scheme, it is a scheme that no longer exists anywhere: apprise 1.12.0 registers no
Microsoft Teams plugin at all. Microsoft retired Office 365 connectors, apprise's `msteams`
went with them, and the replacement is `workflows` (Power Automate). The only registered
scheme containing "teams" is `wxteams`, which is Webex Teams, a different product from a
different vendor.

The thirteen remaining names are correct, and that is the part worth keeping. `discord`,
`slack`, `gotify`, `ntfy`, `ntfys`, `signal`, `matrix`, `json`, `xml`, `form` and `mailto`
really are apprise schemes, because for those services the vendor name and the URL scheme
happen to coincide. A list that is 72% right is not a list anyone re-reads.

`docs/NOTIFICATIONS.md:309` restates the same eighteen strings under a **Fix** heading, so
the troubleshooting page tells a user who hit the rejection to type the exact URL that will
fail next.

## Invariant violated

**An identifier is only as good as the namespace it was drawn from, and a membership test
cannot tell you which namespace you used.** `"telegram" not in ALLOWED_SCHEMES` and
`"tgram" not in ALLOWED_SCHEMES` are the same expression to the interpreter. Nothing in the
type, the test, or the value distinguishes a scheme from a service name, because both are
`str`. The list was validated against human knowledge of which services are supported, which
is a true fact about the wrong vocabulary.

**A gate keyed on the wrong namespace fails in both directions at once, and the two failures
do not look alike.** Every URL that apprise can actually deliver is rejected by the validator
with `Unsupported protocol: tgram`, which reads as "we do not support Telegram" and is exactly
the report that opened this issue. Meanwhile `telegram://`, `pushover://`, `mattermost://`,
`rocketchat://` and `teams://` sail through the security check and then fail one line later
at `temp_apprise.add(url)` with `Failed to add service URL`, a message that names neither the
scheme nor the reason. So the working input gets a confident, specific, wrong explanation, and
the broken input gets a vague one. A user debugging this is steered away from the answer by
both halves.

**The direction that matters for a security control is the permissive one.** This allowlist
exists to bound SSRF: `http`/`https` URLs are subjected to private-IP and DNS checks, and
plugin schemes are trusted because a plugin hardcodes its own endpoint. Five strings sit in a
trust list naming plugins that do not exist. Today they are inert, since apprise refuses to
instantiate them and nothing is dispatched. They are inert by the accident that a *second*
system also rejects them, not by anything this list does, and the day apprise ships a plugin
claiming `teams://` it inherits that trust without anyone revisiting the decision. An
allowlist entry that no code can currently reach is not a safe entry, it is an unreviewed one.

**A vendor's URL scheme is an external contract with its own version history, so freezing it
as a literal is a dependency you did not declare.** `teams` was plausible when it was written
and became unreachable when Microsoft retired the connector and apprise dropped the plugin.
The list carries a pin on apprise's registry with no version bound, no import-time check, and
no test, so the drift is silent in both directions: schemes can disappear, and new ones this
list has never heard of stay blocked.

## Trigger

Configuring or testing any notification service whose apprise scheme differs from its common
name. No unusual configuration, no network conditions, no concurrency. Pasting the URL from
apprise's own documentation, which is what the docs link to, is sufficient and is the reported
path.

## Repro

Clean `python:3.12-slim` container, `--network none`, apprise 1.12.0, the repo's own validator
imported from a full clone at `22b9de908` with `__file__` and sha256 provenance
(`/src/src/local_deep_research/security/notification_validator.py`,
`4b89e69555038e52841c35223660fbca0f7706e837a7d6ccd86d3dda160a0e9a`). Each pair is the same
service written two ways, passed to the real `validate_service_url` and to the real
`apprise.Apprise.instantiate`:

```
URL                          LDR validator   apprise plugin
telegram://bot/chat          PASS            None
tgram://bot/chat             REJECT          NotifyTelegram
pushover://u@t               PASS            None
pover://u@t                  REJECT          NotifyPushover
mattermost://h/t             PASS            None
mmost://h/t                  REJECT          NotifyMattermost
rocketchat://u:p@h/#c        PASS            None
rocket://u:p@h/#c            REJECT          NotifyRocketChat
teams://a/b/c                PASS            None
wxteams://a/b/c              REJECT          NotifyWebexTeams
```

Every row inverts. The validator's rejection message is
`Unsupported protocol: tgram. Allowed: http, https, mailto, discord, sl...`, truncated to the
first five entries, so the four names it would need to show are the ones it cuts.

Enumerating the registry directly in the same container: apprise 1.12.0 registers 210 schemes
across `protocol` and `secure_protocol`; `tgram`, `pover`, `mmost`, `rocket`, `rockets`,
`wxteams` and `workflows` are all present, and `telegram`, `pushover`, `mattermost`,
`rocketchat`, `teams` and `msteams` are all absent.

**What was not verified.** No notification was delivered: there is no Telegram bot token,
Pushover key or Mattermost instance in this environment, so the claim is about which URLs the
validator admits and which apprise will bind to a plugin, not about end-to-end delivery.
`http` and `https` are also absent from apprise's registry and are deliberately excluded from
the defect: LDR allows them for its own generic-webhook path, which is a separate and
intentional branch of the same validator, not a naming error. The reported symptom on
`telegram://` with a real bot token is reached by a different check first, because the token's
colon parses as a port and the URL is rejected as malformed before the scheme test runs; the
scheme defect is isolated above with parser-clean URLs.
