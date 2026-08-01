# A read timeout is reported as a connection failure, so a notification that was delivered is recorded as unsent and the fallback channel sends it again

- **Repo:** caronc/apprise
- **Surface:** `apprise/url.py:113` (`socket_read_timeout = 4.0`) and
  `apprise/plugins/signal_api.py:418-420`
  (`except requests.RequestException`, logging `"A Connection error occured sending"`)
- **Class:** error handling & success reporting
- **Report:** reproduced publicly on
  [issue #1685](https://github.com/caronc/apprise/issues/1685#issuecomment-5151453575),
  closed by the reporter after the diagnosis. No PR: the 4.0s default is deliberate
  (commit `a91064a`, PR #264, which raised it from 2.5 and added the `rto=` override in the
  same change), and the only other candidate change is a wording fix spanning eight plugins,
  which is the maintainer's call rather than a drive-by.

## Root cause

The Signal API POST succeeds and Apprise reports it as a connection failure.

The reporter's server answers the first request after a Gunicorn worker restart in
4.284670901 seconds, by its own access log, and returns 201. Apprise's default read timeout
is 4.0 seconds, so `requests` raises `ReadTimeout` at 4.0s, a quarter second before the
response arrives. `requests.exceptions.ReadTimeout` subclasses `requests.RequestException`,
so it is caught by the generic handler at `signal_api.py:418`, which logs
`"A Connection error occured sending"` and returns `False`.

Measured against a local server that sleeps a fixed interval and then returns 201:

```
delay 4.3s            -> WARNING "A Connection error occured sending 1 Signal API notification(s)."
                         DEBUG   "Read timed out. (read timeout=4.0)"
                         notify() = False, elapsed 4.01s
delay 1.0s            -> INFO    "Sent 1 Signal API notification"
                         notify() = True,  elapsed 1.00s
delay 4.3s, rto=10    -> request_timeout=(4.0, 10.0)
                         notify() = True,  elapsed 4.30s
```

The message was accepted by the server in every one of those runs. Only the third one says
so. The consequence is not a cosmetic log line: `notify()` returning `False` is what a
priority chain reads, so the reporter's configuration fell through to his next transport and
sent the same notification a second time, on a delivery that had already happened.

Two independent facts sit behind this and only one of them is configuration. The 4.0s default
is deliberate and the `rto=` parameter is the sanctioned way to raise it, which resolves this
user's symptom. The classification is separate: nothing in the plugin decides to treat a
timeout as a connection error, the exception hierarchy decides it, because the handler names
only the base class and `ReadTimeout` arrives through inheritance. The wording is a house
template rather than a slip in this plugin, carried by eight transports (pushdeer,
serverchan, viber, parseplatform, signal_api, smseagle, plivo, dingtalk).

## Invariant violated

**A connect-phase failure and a read-phase timeout carry opposite information about whether
the side effect happened, and collapsing them destroys exactly the fact the caller needs.**
A connection error means the request never reached the server, so the operation did not
occur and retrying is safe. A read timeout means the request was written and the client
gave up waiting, so the operation may well have completed, and retrying duplicates it. These
are the two halves of the retry-safety decision. `requests` models them as distinct classes
for that reason, and a handler that catches `RequestException` erases the distinction without
ever mentioning it.

**The collapse is invisible in the code that suffers from it.** The catch site names one
symbol and behaves correctly for the case its message describes. The defect lives in the set
of exception types that reach it, which is declared somewhere else entirely, in a library's
class hierarchy. Reading the handler cannot reveal the problem; enumerating the subclasses of
the caught type can. That makes an over-broad `except` a different hazard from a swallowed
error: nothing is hidden, the report is simply about the wrong event.

**Reporting a completed operation as failed is more dangerous than the reverse in any system
with a fallback path, because the fallback is a retry that no one calls a retry.** A false
success stops the pipeline and loses a message. A false failure activates every recovery
mechanism the operator built, and those mechanisms are designed to be reliable, so they will
faithfully deliver the duplicate. For a notification library the fallback is the priority
chain, and it turns one misclassified exception into two messages to a human. Any component
returning a boolean delivery result is making a machine-readable claim about a side effect,
and it should return `False` only for outcomes it can distinguish from "already done".

**A default that is documented and overridable is still a threshold, and thresholds decide
correctness at their boundary.** 4.0s is a reasonable value and `rto=` is a real answer for
this user. Neither changes what happens 0.28 seconds past the line: the answer is not "slower
than we would like", it is "unreachable", and that is a claim about a different thing.

## Trigger

Any Signal API request that takes longer than the 4.0s default read timeout, on the shipped
configuration, with no `rto=` override. A cold first request is the natural instance: the
reporter sees it on the first notification after a Gunicorn worker restart, where connection
setup and process warmup land on one request. The failure is transient by construction, which
is why it reads as an intermittent connectivity problem rather than as a timeout, and why the
duplicate on the fallback channel looks unrelated to it.

## Repro

Clean `python:3.12-slim` container at `ea071dbe`. A local `http.server` sleeps a configurable
interval and then returns 201, with an Apprise `signal://` URL pointed at it, so no Signal
credentials and no network access are involved and the only variable is response latency. The
three runs above are that server at 4.3s, at 1.0s, and at 4.3s with `?rto=15`, reading back
`notify()`'s return value, the emitted log lines and the elapsed time.

The history was read on a full clone with both probe strings escaped for regex metacharacters
and cross-checked with `grep -cF`, so a zero-hit result would have been treated as a broken
probe rather than as an absence of history. `socket_read_timeout = 4.0` returns one commit,
`a91064a` (PR #264, 2020-07-31), whose diff raises the value from 2.5 and introduces `rto=`
and `cto=` together, which is what establishes the default as a considered choice. The log
wording returns `41fe862` (PR #568) for this plugin and the same string across seven others.

**Not verified:** no real Signal API server and no Gunicorn worker were involved. The
4.284670901s figure is from the reporter's own access log, not from anything measured here,
and the claim reproduced here is the narrower one that any response slower than the read
timeout produces this classification. The priority-chain fallthrough is the reporter's
configuration, described from his report rather than executed.

Verified 2026-08-01 at `ea071dbe`; the two cited lines were re-read at `26c88f3c` and are
unchanged. The issue was closed as completed by its author, who said he would raise the
timeout on his side. No code changed upstream, so the classification described here is still
what ships.
