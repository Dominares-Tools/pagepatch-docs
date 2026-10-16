# Changelog

## 1.1.2

README only. No code change.

- The release README opens with fill mode and its sample command. Version number only otherwise.

## 1.1.1

Fixes found in review of fill mode.

- `fill --state` with `-o` is now a usage error (exit 2). The state describes the rewritten page, but `-o`
  leaves the input as it was, so run 2 saw every part as hand-edited and wrote the old content back.
  Use `-i`.
- `fill --state` saves the fingerprints whenever the page is rewritten, also when another entry was
  refused. Before, the good entries kept stale fingerprints and froze as "edited by hand".
- A bind inside a region no longer freezes the region: the region's fingerprint is taken after the binds
  are written, and a bind reset by a region write in the same run is not reported as hand-edited.
- A text bind on the same element as a region being filled, or around one, is refused instead of
  silently overwriting the region.
- Region markup: `/>` is honoured only on void and svg or math elements (`<div/>` is refused as an open
  div); `<plaintext>`, `<xmp>`, `<noembed>` and `<noframes>` are refused.
- Attribute writes: `http-equiv` is never added to a `<meta>`, a stylesheet or preload `rel` is never set
  on a `<link>`, and SVG animation elements take no attribute writes.
- Row templates: a `{name}` or `{name|url}` placeholder in a tag outside a quoted attribute value is a
  `SpecError` (`check_template`, also run when there are no rows).
- `pagepatch check` takes `--today`; the stdin guard covers every input of every command.
- `lost_visible_text` is reimplemented on BeautifulSoup rather than taken line for line from the script
  it came from: it compares text nodes and ignores `<noscript>` and `<template>` text.

## 1.1.0

New: fill mode. Fill marked parts of a hand-made page from data, byte for byte, and refuse any
write that changes anything else. Edit mode is unchanged.

- `pagepatch fill page.html spec.json`: fills `data-pagepatch-region` blocks (markup, plain text,
  or a row template) and `data-pagepatch-bind` elements (text and attributes) from a JSON spec.
  Only the marked bytes change; the doctype, whitespace, void tags, inline scripts and styles and
  CRLF line endings stay exactly as authored.
- An independent outside-bytes check (`check_write`, `pagepatch check`), computed from the spec, runs
  before every write; it also refuses to shrink the overrides `<style>` block below half.
- Safety on what is written: region markup must be self-contained, script and style regions cannot be
  closed from inside, event handlers and script URLs are never written, attributes of executable
  elements are never written, nested regions are refused. `js_json()` makes data safe inside a script.
- Hand-edit detection: `--state` keeps a fingerprint per part; a part edited since the last sync is
  skipped unless `--always-refresh`. Missing markers are always reported.
- `pagepatch mark` installs the markers by CSS selector: idempotent, all-or-nothing, self-checked.
- `pagepatch merge` carries a live page's regions, sections and overrides block into a new template,
  refuses ambiguous or duplicated parts, and refuses to drop visible live text unless told to.
- `pagepatch regions` lists the markers on a page.
- Library: `fill`, `plan_fill`, `check_write`, `outside_hash`, `install_markers`, `merge_preserving`,
  `lost_visible_text`, `find_elements`, `set_inner_html`, `set_attr`, `region_fingerprint`,
  `render_rows`, `js_json`, `url_is_safe`, `read_page`, `write_page` (atomic, compare-before-replace),
  `load_state`, `save_state`.
- Examples: `examples/sync_page.html` and `examples/sync_spec.json`.
- Date windows: rows and regions with `active_from` / `active_until` are dropped or cleared as of `--today`
  (`today=` in Python), so a dated notice removes itself.
- Changelog wording for 1.0.1 tidied.

## 1.0.2

Documentation and packaging only. The code is unchanged.

- The README now opens with a one-line description and a three-line example you can paste,
  and the licence and access notes moved to the end.
- The package version (`pip show pagepatch`, `pagepatch.__version__`) now matches the release tag.
  Release 1.0.1 reported 1.0.0.

## 1.0.1

Licence and contact details only. The code is unchanged.

- Licence text updated (the commercial licence, version 1.3).
- One public support address in the licence and the README.

## 1.0.0

First release.

- Directive grammar (`CSS_OVERRIDE`, `HTML_REPLACE`, `HTML_SET_TEXT`, `HTML_INSERT_BEFORE`,
  `HTML_INSERT_AFTER`, `HTML_INSERT_INSIDE`, `HTML_DELETE`, `PATCH`, `FULLHTML`) with a
  forgiving parser: code fences, missing closers, bracketed and unquoted selectors.
- `apply()`: each directive applied and reported on its own, good edits kept when others
  fail, closest real selectors offered on a miss, a selector matching several elements
  refused with one unique selector per match (`Config.multi_match="first"` to edit the
  first instead), CSS spliced into one overrides block without re-serializing the page,
  `!important` promotion when the page forces a property, marker-drop warnings, ids for
  new sections, overrides carried through a rebuild. HTML edits keep attribute order.
- Safety enforced by the engine: script vectors and off-site stylesheet links stripped from
  model markup; every HTML directive and PATCH checked by what it added to the page (a
  script, handler, script URL, frame, off-site stylesheet or freezing rule is refused); no
  edits inside or removal of `<script>`/`<style>`; `@import` refused; page-freezing CSS
  refused through every directive (escapes resolved, selector subject resolved by the CSS
  engine, fails closed); cut-off rebuilds rejected; custom guards.
- Page map and outline rows in which every selector matches exactly one element; focused
  edit context with the element's own cascade.
- Checks derived from what each applied edit put on the page, a static verifier, and a
  browser verifier (`pagepatch/js/verify.mjs`) with matching retry wording.
- `propose()`: a provider-neutral prompt, apply, verify and self-correct loop for any
  `prompt -> reply` callable, with prompt templates; `examples/chat_http.py` wires it to a
  chat API over HTTP with the standard library (configurable token parameter, error bodies
  shown). `examples/page.html` is a sample page for the quickstart.
- Style-panel helper with a property allowlist, stale override pruning, and the
  `pagepatch apply | map | context` command line, which always writes the page.
