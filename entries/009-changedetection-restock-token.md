# A notification token valid per-watch crashes rendering system-wide

- **Repo:** dgtlmoon/changedetection.io
- **Surface:** `notification_service.py::NotificationContextData` default token dict
- **Class:** template & token rendering
- **Fix:** [PR #4249](https://github.com/dgtlmoon/changedetection.io/pull/4249) (merged)

## Root cause

The `restock` token was only injected into the notification context for
restock-type watches. A global notification body using `{{ restock.price }}`
raised a jinja2 `UndefinedError` at send time for every non-restock watch,
and was rejected at save-time validation unless a restock watch happened to
exist.

## Invariant violated

A token accepted in per-watch settings must validate in system-wide settings
independent of watch inventory, and must never crash rendering for a watch
type that doesn't populate it. Safe-empty beats absent.

## Trigger

Global notification body referencing `restock.*` while any non-restock watch
fires.

## Repro

Reproduced on HEAD with a jinja repro; fix is a one-line safe-empty
placeholder in the default dict plus a pytest modeled on the repo's existing
token tests. Maintainer: "good enough fix for now" — the deeper refactor is
theirs to schedule.
