# Watch URL with Jinja tokens shown raw and non-clickable on the diff page

- **Repo:** dgtlmoon/changedetection.io
- **Surface:** diff/preview page URL rendering (`processors/.../difference.py`)
- **Class:** template & token rendering
- **Fix:** [PR #4256](https://github.com/dgtlmoon/changedetection.io/pull/4256) (issue [#3776](https://github.com/dgtlmoon/changedetection.io/issues/3776))

## Root cause

The diff page passed the raw watch URL *template* into the view, which rendered
it directly as an `href`. A URL containing Jinja syntax, e.g.
`{% now 'utc','%Y' %}.csv`, was displayed unparsed and non-clickable. The
`Watch.link` property already performs the Jinja render plus safe-URL
validation, but the diff render path bypassed it and used the stored template
string. The same raw-template pattern appears on the preview page and the image
processor.

## Invariant violated

A value that has a canonical rendered-and-validated form must be displayed in
that form, never as its raw template. Once a model exposes "the safe, rendered
URL" (`Watch.link`), every consumer that shows a URL must go through it;
reaching past the accessor to the raw field re-opens the exact hole the
accessor was built to close.

## Trigger

A watch whose URL contains Jinja template syntax, viewed on the diff (or
preview) page.

## Repro

Flask test client: quick-add a watch whose URL contains `{% now 'utc','%Y' %}`,
GET the diff page, assert the rendered year appears inside the `href` and the
raw `{% now` token is absent. Fails on HEAD, passes once the render call uses
`Watch.link`.
