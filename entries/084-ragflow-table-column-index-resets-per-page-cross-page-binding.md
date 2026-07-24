# A table's column-group id resets per page, but the column list spans the document, so a later page's cell binds to an earlier page's column by x-distance alone

- **Repo:** infiniflow/ragflow
- **Surface:** `deepdoc/vision/recognizer.py`, `Recognizer.find_horizontally_tightest_fit`
  (column selection), with the ids assigned in `deepdoc/parser/pdf_parser.py`
- **Class:** indexing, ordering & counting contracts
- **Fix:** [PR #17282](https://github.com/infiniflow/ragflow/pull/17282)
  (in review; issue [#17199](https://github.com/infiniflow/ragflow/issues/17199))

## Root cause

When assembling a table, each cell picks its column by
`find_horizontally_tightest_fit`, which filters candidate column boxes on their
`layoutno` and then takes the horizontally closest one by x-distance. The column id
is `layoutno = f"table-{page_table_index}"`, and `page_table_index` resets on every
page, so `"table-0"` names a different physical table on each page. The `clmns` list
passed at the call site spans the whole document. Together these mean a cell on page
2 whose table is `"table-0"` is matched against every `"table-0"` column in the
document, including page 1's, and since the tie-break is x-distance alone, a page-1
column that happens to be horizontally closer wins. The result is a table whose
cells bind to another page's column set: wrong column count and misplaced cell
coordinates.

The sibling row and header matches survive the same reset because they compare by
geometric overlap on `top`/`bottom`, which are page-cumulative and so never overlap
across pages. Only the column match relied on a per-page-reset id plus a pure
x-distance tie-break, with no vertical constraint to keep it on its own page.

## Invariant violated

An id that resets its counter within each page is unique only per page, so any
consumer that compares such ids across a document-spanning collection must also
carry the page (or an already-page-safe coordinate) into the match, or the id
collides across scopes. A local index is a valid key only inside the scope that
resets it; joining on it across a wider scope silently unions distinct entities that
share the reset value. When the discriminator (`layoutno`) is scope-local and the
candidate pool (`clmns`) is global, the missing predicate is the scope itself: the
match must additionally require the candidate to share the cell's page, here
expressible as shared vertical extent because `top`/`bottom` are page-cumulative at
this point. The neighboring row/header paths were correct not by intent but because
their geometric overlap already encoded the page; the column path used distance
alone and lost it.

## Trigger

A multi-page PDF with a table on more than one page (in particular tables that reuse
the same per-page table index, `"table-0"` on each page) where a column on an
earlier page is horizontally closer to a later page's cell than that cell's own
column. The later page's cells bind to the earlier page's columns, producing a wrong
column count and shifted cell coordinates. Single-page documents, and multi-page
documents whose per-page columns never align horizontally, do not surface it.

## Repro

Docker-verified this session in a clean container against HEAD `4a2564c`, driving
the real `find_horizontally_tightest_fit` (not a re-implementation): a page-2 cell
bound to a page-1 column before the fix and to the correct page-2 column after. The
fix requires the candidate column to share vertical extent with the cell; since
`top`/`bottom` are page-cumulative here, a column from another page never overlaps
and is rejected, while within a page the winning column is unchanged. The added unit
test fails on `main` and passes on the branch, the existing table-coordinate suite
still passes, and `ruff check` is clean. Not exercised: the full end-to-end PDF
parse needs the layout models (GPU/model download), so the regression is pinned at
the unit level on the exact function that decides column binding.
