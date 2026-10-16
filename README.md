# pagepatch — Let an AI make small, checkable edits to a page

**Paid tool: $99 once (plus sales tax where it applies), then a private GitHub repository you're invited to, with every update. Buy: https://dominares.org/tools/pagepatch**

Commercial license v1.3 — see https://dominares.org/license · Support: support@dominares.org

pagepatch sits between a model's reply and your saved page. The model writes small edits, pagepatch applies each on its own, refuses the unsafe ones, checks the rest landed and tells the model exactly what to fix. A second mode, fill, puts data from a JSON file into the parts of a hand-made page you mark, byte for byte, and refuses any write that changes anything else.

## Get it

Buy pagepatch at https://dominares.org/tools/pagepatch: one payment of $99, plus sales tax where it applies, and every future update. After checkout you are invited to a private GitHub repository with the source and every version published to it. The repository includes the Python engine, the pagepatch command (apply, map, fill, mark, merge and check) and a browser verifier.

Requirements: Installs BeautifulSoup and soupsieve. No AI vendor SDK is needed.

This page documents version 1.1.2. Install commands and file paths in the documentation below run inside the private repository you are invited to; the source is not in this repository.

## Documentation

### What it does

pagepatch has two ways of changing a page, and both leave the rest of the page alone.

**Fill mode** (new in 1.1, no AI involved). You mark the parts of a hand-made page that change
(`data-pagepatch-region` for a block, `data-pagepatch-bind` for a small element), keep the data in a
JSON spec, and `pagepatch fill` splices the new bytes in. It is a different job from a template
engine: the page is the template and stays a normal HTML file. Before anything is saved, a second,
independent check proves that the bytes outside the filled parts are identical to the input, and the
write is refused if they are not. It also skips parts someone edited by hand, carries a live page's
regions and styles into a new template, and installs the markers for you. See
[Fill a page from data](#fill-a-page-from-data).

**Edit mode.** A model replies with a small edit language; pagepatch applies each edit on its
own, reports each on its own, and checks that it landed. See [Edit a page with a model](#edit-a-page-with-a-model).

### Edit a page with a model

When you ask a model to change a web page, the easy way is to send it the whole HTML file and
take back a whole new file. That is slow and expensive on a real page, and it goes wrong in
quiet ways: the model drops a script, rewrites a section you did not mention, or returns a
file that was cut off at the token limit.

pagepatch gives the model a small edit language instead, and does the editing itself:

```
Making the headline shorter and the order button green.
[HTML_SET_TEXT selector="#hero h1"]Fresh bread, daily.[/HTML_SET_TEXT]
[CSS_OVERRIDE]
.nav-order { background: #0a7d4f; }
[/CSS_OVERRIDE]
```

Each edit is applied on its own and reported on its own. A bad selector fails that one edit,
not the batch, and comes back with the closest real selectors on the page so the model can
correct itself in one more round. A selector that matches several elements edits none of
them and comes back with one exact selector per match, so the wrong element is never
changed. CSS edits are added to one `<style>` block without rewriting the rest of the page.
Edits that would add JavaScript, or freeze or blank the page in the known ways, are refused
or stripped. After applying, pagepatch checks that each change actually landed.

### Who it is for

Developers building an "edit my site with AI" feature: a site builder, a CMS, an internal
tool that lets staff change a landing page, an agent that maintains static pages. You bring
the model and the storage; pagepatch is the part between the model's reply and the saved page.

Fill mode is for anyone who keeps a hand-made static page and a source of data for parts of it
(a menu, an events list, prices, a status line, a count) and wants the data on the page without
a CMS or a site generator, and without a sync job that can quietly damage the rest of the file.

It is a Python library (with a small JavaScript module for checking results in a browser)
and a command line tool. It does not call any AI provider itself and has no vendor SDK
dependency.

### Install

*The install commands are in the README of the private repository you are invited to when you buy. The notes on them are kept here.*

### 60-second quickstart

The repository ships a sample page. From the same folder, copy it next to you:

```
cp examples/page.html page.html
```

Save a model reply (or hand-written edits) to `edits.txt`:

```
[HTML_SET_TEXT selector="#hero h1"]Fresh bread, daily.[/HTML_SET_TEXT]
[CSS_OVERRIDE].nav-order { background: #0a7d4f; }[/CSS_OVERRIDE]
[HTML_SET_TEXT selector="h2"]Opening hours[/HTML_SET_TEXT]
[HTML_DELETE selector=".promo-banner"][/HTML_DELETE]
```

Apply it:

```
$ pagepatch apply page.html edits.txt -o new.html
ok   html_set_text #hero h1: Set text on 1 element (#hero h1)
ok   css_override: Applied CSS override
FAIL html_set_text h2: selector matched 3 elements, so nothing was edited: use the unique selector for the one you mean (offered below), or add all="true" to edit every one
     try: #menu h2, #hours h2, #contact h2
FAIL html_delete .promo-banner: selector matched 0 elements
     try: .topbar, .btn-primary, #main-content, .band, .brand, .price
```

The two good edits are in `new.html`; the exit code is 1 because two edits failed, and each
failure says what to use instead. See what a model would be told about the page (every
selector in the map matches exactly one element):

```
$ pagepatch map page.html
header.topbar  ·  header  ·  "Harbor Street Bakery Menu Hours Contact Order ahe…"
    a.brand  ·  link  ·  "Harbor Street Bakery"
    nav.nav-links  ·  nav "Primary"  ·  "Menu Hours Contact"
    a.nav-order  ·  link  ·  "Order ahead"
#main-content  ·  main  ·  "Bread baked before sunrise. Sourdough, pastries a…"
  #hero  ·  section  ·  "Bread baked before sunrise. Sourdough, pastries a…"
      h1  ·  heading  ·  "Bread baked before sunrise."
        #hero a.btn-primary  ·  link  ·  "Order ahead"
        a.btn-ghost  ·  link  ·  "See the menu"
  #menu  ·  section  ·  "Today's menu Country sourdoughTwo-day ferment.$9 …"
      #menu h2  ·  heading  ·  "Today's menu"
...
```

The same from Python:

```python
import pagepatch

html = open("page.html", encoding="utf-8").read()
result = pagepatch.apply(html, open("edits.txt", encoding="utf-8").read())

for row in result.summary:
    print(row["ok"], row["kind"], row["selector"], row["note"], row["candidates"])
open("new.html", "w", encoding="utf-8").write(result.html)
```

And the whole loop with a model (any function that takes a prompt string and returns the
reply string):

```python
out = pagepatch.propose(my_llm, html, "Make the order button green", max_rounds=3)
if out.ok:
    save(out.html)
else:
    print(out.unresolved, out.failed_checks)
```

### The edit language

A reply is prose plus any number of bracketed directives. Everything outside the brackets
is kept as `commentary`.

| Directive | What it does |
|---|---|
| `[CSS_OVERRIDE] rules [/CSS_OVERRIDE]` | Appends CSS rules to `<style id="pagepatch-overrides">` (created before `</head>` if missing). The rest of the page is not touched. |
| `[HTML_REPLACE selector="…"] markup [/HTML_REPLACE]` | Replaces the element (its full outer HTML). |
| `[HTML_SET_TEXT selector="…"] text [/HTML_SET_TEXT]` | Replaces the element's contents with text (markup is escaped). |
| `[HTML_INSERT_BEFORE selector="…"] markup [/HTML_INSERT_BEFORE]` | Inserts a sibling before the element. |
| `[HTML_INSERT_AFTER selector="…"] markup [/HTML_INSERT_AFTER]` | Inserts a sibling after the element. |
| `[HTML_INSERT_INSIDE selector="…" position="start\|end"] markup [/HTML_INSERT_INSIDE]` | Inserts as the first or last child (default `end`). |
| `[HTML_DELETE selector="…"][/HTML_DELETE]` | Removes the element. |
| `[PATCH]FIND: old REPLACE: new[/PATCH]` | Literal text swap of the first occurrence. If FIND is not found and REPLACE is plain CSS, it is applied as a CSS override instead. |
| `[FULLHTML] document [/FULLHTML]` | Replaces the whole page. Rejected unless it is a complete document. |

HTML directives edit one element; add `all="true"` to edit every match. When a selector
matches more than one element and `all` is not set, nothing is edited: the result has
`multi_match=True` and `candidates` holds one selector per match that matches only that
element (`#menu h2`, `#hours h2`, `#contact h2`), so the model can pick the one it meant.
Set `Config(multi_match="first")` to edit the first match with a warning instead.

No HTML directive may write inside a `<script>`, `<style>`, `<noscript>` or `<template>`,
replace or delete one (or a stylesheet `<link>`), or replace, clear or delete `<html>`,
`<head>` or `<body>` (that is what FULLHTML is for). `HTML_SET_TEXT` may change `<title>`
and `<textarea>` text.

The parser is forgiving, because models are: markdown code fences are stripped, a missing
`[/NAME]` closer ends at the next directive, selectors may contain brackets inside quotes
(`selector="[data-x='y']"`), and unquoted attributes work (`selector=.title`).

### What happens to each edit

`pagepatch.apply(html, directives)` returns an `ApplyResult`:

- `html`, `changed`: the new page and whether it differs.
- `results`: one `OpResult` per directive, in order, with `ok`, `note` (what was done) or
  `reason` (why not), `candidates` (real `#id`/`.class` selectors closest to a selector that
  matched nothing, or one unique selector per match for a selector that matched several),
  `match_count`, `multi_match`, `dropped_markers`.
- `failed`, `applied`, `had_failures`, `summary` (JSON-ready rows), `commentary`.

Along the way pagepatch also:

- **Wins the cascade for you.** If the page already declares a property with `!important`
  on the same element (through the same selector or a different one), the incoming
  declaration is promoted to `!important`. Otherwise the edit "applies" and nobody sees it.
- **Warns about CSS that styles nothing.** A rule whose selector matches no element is still
  applied (it may be for a class added later by JavaScript) but comes back with a warning
  and candidates.
- **Keeps your markers.** Any `data-pagepatch-*` attribute (plus any you list in
  `Config.marker_attrs`) is treated as something your own code relies on. If an
  `HTML_REPLACE` or `HTML_DELETE` removes one, the result says which.
- **Gives new sections an id.** After an edit that adds markup, every `<section>` directly
  inside `<main>` or `<body>` without a unique id gets one from its heading
  (`<h2>Gift Cards</h2>` becomes `id="gift-cards"`), so it can be linked as `#gift-cards`.
  The ids added are listed in `ensured_ids`.
- **Carries your CSS through a redesign.** A `FULLHTML` rebuild keeps the accumulated
  overrides block, and re-adds external `<script src>` tags and Google Fonts links the old
  page had and the new one dropped.

### Safety

These rules are applied by the engine to every directive, whatever the prompt said. Your
application still decides what gets published; see "Honest limits".

- **No new JavaScript.** Inserted markup and FULLHTML documents are stripped of `<script>`,
  `<iframe>`, `<frame>`, `<object>`, `<embed>`, `<applet>`, `<noscript>`, `<base>`,
  `<meta http-equiv>` (a refresh redirect), every `on*=` handler and every `javascript:`,
  `vbscript:` and `data:` URL, including ones split by tabs or newlines the way browsers
  ignore them. Scripts, `<base>` and `<meta http-equiv>` tags the page already had are kept
  (same `src`, the same inline text, or the identical tag). To allow new first-party bundles, set
  `Config.trusted_script_prefixes=("/static/",)`.
- **Every edit is checked by its result, too.** After each HTML directive and each PATCH,
  the new page is compared with the old one. If it gained a script, an event handler, a
  script URL, a frame, an off-site stylesheet or a page-freezing rule, the edit is refused
  and the page is left as it was. A PATCH may not touch the inside of a `<script>`,
  `<style>`, `<noscript>` or `<template>`, and HTML directives may not target one (see
  "The edit language").
- **No CSS from elsewhere.** `@import` is refused in CSS_OVERRIDE and in inserted `<style>`
  (font CSS from `fonts.googleapis.com` excepted), and a FULLHTML rebuild may only keep the
  imports the page already had. A `<link rel="stylesheet">` (or `import`, `preload`,
  `modulepreload`, `prefetch`, `prerender`) in model markup is removed unless the page
  already has the identical tag, it points at the page's own site (a relative URL), or it is
  font CSS from `fonts.googleapis.com`.
- **Page-freezing CSS is refused.** Rules on `html`, `:root` or `body` that set `overflow`,
  `position: fixed`, `height`, `inset`, `display: none`, `visibility: hidden` or
  `collapse`, `contain: strict`, or an `opacity`, `filter: opacity()`, `scale` or
  `transform: scale()` at or near zero (or a value that cannot be read, such as `var()`) are
  refused, through every directive: CSS_OVERRIDE, a `<style>` inside inserted markup, a
  `style=` attribute on `<html>`/`<body>`, a PATCH, and FULLHTML. Property names are read
  after CSS escapes are resolved, so `\6f verflow` is `overflow`. Which element a rule
  styles is decided by running the selector against a probe document, so `:is(html)`,
  `body.dark`, `html > body`, `[lang]` and escaped selectors are all caught; a selector
  that cannot be evaluated is refused. A whole-document rebuild uses a narrower rule (it may
  set `height` and `overflow-x: hidden`, which real pages do). This is a list of the known
  ways to freeze or blank a page, not a proof that no other CSS can.
- **No broken style block.** CSS containing `</style>` or `<script` is refused.
- **No half pages.** A FULLHTML body without `<html`/`<!doctype` and `</html>` (a reply cut
  off at the token limit) is rejected and the page is left as it was.
- **Your own rules.** `Config.guards` takes functions that see each directive before it is
  applied and return a refusal reason, or None. Pair each with a sentence in
  `extra_rules=` so the model knows the rule too.

```python
def keep_prices_generated(d):
    if d.kind.startswith("html_") and ".price" in d.selector:
        return "prices come from the menu system; change them there"

config = pagepatch.Config(guards=(keep_prices_generated,))
out = pagepatch.propose(my_llm, html, request, config=config,
                        extra_rules="Never edit elements with class .price.")
```

`pagepatch.count_executable_vectors(html)` is also useful on any path that saves a whole
page: if the count goes up, the save added executable content.

`examples/chat_http.py` sends `max_tokens` by default; for endpoints that want
`max_completion_tokens`, pass `token_param="max_completion_tokens"`. A `rebuild=True`
redesign returns the whole page, so raise `max_tokens` to at least the page's length in
characters divided by three, or the reply is cut off and rejected as an incomplete document.

### Connecting a model

`propose(llm, html, instruction, ...)` runs the loop:

1. Build a prompt: the instructions and edit language, your `notes`, a focused context and
   the request. The focused context is the page map, plus (when `selector` is given) the
   element's markup, its ancestors and siblings, and every CSS rule on the page that targets
   it, including overrides. The whole page is only sent for a rebuild.
2. Call `llm(prompt)` and apply the reply.
3. Derive checks from the directives that applied and run `verify` on the new page.
4. If a directive failed or a check failed, call the model again with its previous reply,
   the failures and the real candidate selectors. Its corrections are applied on top of what
   already worked. Repeat up to `max_rounds`.

`llm` is any `Callable[[str], str]`. `examples/chat_http.py` builds one for the two common
HTTP request shapes using only the standard library:

```python
import sys
sys.path.insert(0, "examples")
from chat_http import openai_style, messages_style

llm = openai_style("https://api.example.com/v1", api_key, "your-model-name")
# or: llm = messages_style("https://api.example.com/v1", api_key, "your-model-name")
out = pagepatch.propose(llm, html, "Add a gift card section after the menu")
```

If you already use an SDK, wrap it the same way:

```python
def llm(prompt: str) -> str:
    return my_client.complete(prompt)   # whatever your client's call is
```

`ProposeResult` has `html`, `changed`, `ok`, `rounds` (each with its prompt, reply, results
and failed checks), `results`, `checks`, `failed_checks`, `unresolved` (failures no later
round answered), `commentary`, `ensured_ids` and `error` (set if a retry call raised; the
progress before it is kept).

To build prompts yourself: `pagepatch.build_prompt(...)`, `pagepatch.heal_instructions(...)`,
and the templates in `pagepatch.prompts`.

### Checking the result

`pagepatch.build_checks(directives, results=result.results)` turns edits into checks such as
`{"kind": "text", "selector": "#hero h1", "expected": "Fresh bread, daily."}`,
`{"kind": "css", "selector": ".nav-order", "property": "background-color", "expected": "#0a7d4f"}`
or `{"kind": "exists", "selector": "p.gift", "text": "Gift cards at the counter."}`.
With `results`, directives that failed are skipped; without it, pass only the directives
that applied. An insert or replace is checked by what it put on the page (its root
element's id, or its tag and class holding its text), not by the selector it was aimed at,
so a replace that renames a class still verifies. Checks are only made for values that can
be compared reliably (a `1.2rem` font size or a gradient background is skipped rather than
reported as a failure).

- `pagepatch.verify_html(html, checks)` checks without a browser: text, existence, absence,
  the overrides block, and for CSS that the element exists and a rule setting the value is
  on the page. It returns the failed checks with an `actual` field.
- `pagepatch/js/verify.mjs` (installed inside the package) checks in a browser with
  `getComputedStyle`, which is the only way to prove a CSS rule won the cascade:

```js
import { runChecks, buildHealText } from "./verify.mjs";
iframe.srcdoc = newHtml;
iframe.onload = () => {
  const { failed } = runChecks(iframe.contentDocument, checks);
  if (failed.length) retryWith(buildHealText(failed));
};
```

To use browser checks inside `propose`, pass `verify=` a function `(html, checks) -> failed`
that renders the page (for example with a headless browser) and runs `runChecks`.
`verify.mjs` also exports `uniqueSelector(element)` for click-to-select editors.

### Other tools in the box

- `pagepatch.build_page_map(html)` / `build_outline(html)`: the page map as text for a
  prompt, or as rows (`selector`, `role`, `text`, `depth`, `tag`, `stable`) for a UI. Every
  selector matches exactly one element: a repeated `h2` is offered as `#hours h2`, and an
  element with nothing to anchor on gets a position path (`#menu > div > article:nth-of-type(2)`).
- `pagepatch.unique_selector(element, soup)`: the same unique selector for one parsed
  BeautifulSoup element, for a click-to-select editor on the Python side.
- `pagepatch.build_edit_context(html, selector=..., page_map=...)`: the focused context on
  its own, with stats on how much smaller it is than the page.
- `pagepatch.style_directive(selector, {"color": "#fff", "font-size": "18px"})`: a
  validated CSS_OVERRIDE for a style panel, no model needed. Properties are an allowlist
  (color, background-color, font-size, font-weight, line-height, letter-spacing, text-align,
  text-transform, padding, margin, border-radius, max-width, opacity, display) and values
  are pattern-checked; anything else raises `StyleRefused` with a readable reason.
- `pagepatch.prune_overrides(html, rules)`: remove override rules of shapes you declare
  (`StaleRule(name, why, selector_predicate, body_predicate)`) once your stylesheet no
  longer needs them, with any `@keyframes` they alone used.
- `pagepatch.sanitize_html(fragment)`, `overrides_css(html)`, `ensure_section_ids(html)`,
  `lint_css(css)`, `lint_document_css(html)`.

### Configuration reference

`pagepatch.Config` (pass as `config=` to `apply`, `propose`, `build_page_map`,
`build_outline`, `build_edit_context`, `build_checks`):

| Field | Default | Meaning |
|---|---|---|
| `overrides_id` | `"pagepatch-overrides"` | id of the `<style>` block CSS edits go into. |
| `marker_attrs` | `()` | Extra attribute names treated as markers, on top of every `data-pagepatch-*` attribute. |
| `trusted_script_prefixes` | `()` | `<script src>` prefixes model markup may add (for example `("/static/",)`). |
| `ensure_section_ids` | `True` | Give new top-level sections a unique id. |
| `guards` | `()` | Functions `directive -> reason or None` run before each directive. |
| `multi_match` | `"refuse"` | A selector matching several elements without `all="true"`: `"refuse"` edits nothing and offers a unique selector per match; `"first"` edits the first with a warning. |

`propose()` keyword arguments:

| Argument | Default | Meaning |
|---|---|---|
| `max_rounds` | `3` | Model calls at most, including the first. |
| `selector` | `""` | The element the request is about, if the user picked one. |
| `rebuild` | `False` | Ask for a whole-page FULLHTML redesign (sends the whole page). |
| `notes` | `""` | Facts the model must use rather than invent (links, colors, hours). |
| `extra_rules` | `""` | Your site rules in words, added to the prompt's safety section. |
| `verify` | `verify_html` | `(html, checks) -> failed checks`, or `None` to skip. |
| `budget_chars` | `60000` | Cap on the focused context's size. |
| `config` | `Config()` | As above. |

Command line:

```
pagepatch apply PAGE EDITS [-o OUT | -i] [--json] [--overrides-id ID] [--marker ATTR]
                           [--trust-script PREFIX] [--no-section-ids] [--no-verify]
pagepatch map PAGE [--json] [--limit N] [--marker ATTR]
pagepatch context PAGE [--selector SEL] [--overrides-id ID] [--marker ATTR]
```

`PAGE` or `EDITS` may be `-` for stdin. `apply` always writes the page, changed or not: to
`-o OUT`, over the page file with `-i` (only when something changed), or to stdout. So
`pagepatch apply page.html edits.txt > new.html` never leaves an empty or stale file. It
prints one line per directive to stderr, and exits 0 when everything applied and verified,
1 when anything failed or there was nothing to apply, 2 on a file error. With `--json`,
stdout carries the JSON report instead of the page, so use `-o` or `-i` with it to keep
the new page.

### Honest limits of edit mode

- **HTML edits re-serialize the page** through Python's `html.parser`. Attribute order,
  bare boolean attributes (`defer`) and `<br>` are kept, but attribute quotes become double
  quotes, `<!doctype html>` becomes `<!DOCTYPE html>` followed by a blank line, `&` in an
  attribute is written `&amp;`, and (with recent beautifulsoup4 releases; 4.15 tested)
  indentation made only of whitespace between tags is reduced to a newline. The page
  renders the same, but a byte diff shows these. CSS-only edits and PATCH never
  re-serialize.
- **The freeze lint is a denylist.** It refuses the known ways to freeze or blank a page
  (listed under "Safety"); CSS it does not know about can still hide content. Keep a
  preview before publishing.
- **A PATCH is compared by text, too.** A PATCH whose REPLACE adds text such as
  `javascript:` or `onclick=` is refused even when it is only prose; use HTML_SET_TEXT for
  text that mentions them.
- **Static checks cannot see computed styles.** `verify_html` confirms a rule exists for an
  element that exists; only the browser module can prove it won the cascade.
- **Cascade promotion only knows `!important`.** A plain rule that loses on specificity is
  not rewritten; the model is shown the competing rules so it can pick a stronger selector,
  and a browser check will catch it.
- **The CSS handling is not a full CSS parser.** It splits flat rules and handles one level
  of `@media` nesting; exotic syntax is passed through unchecked rather than refused (except
  for the freeze lint, which refuses what it cannot evaluate).
- **`data:` URLs are stripped** from `src`/`href` in model markup, including images.
  Upload images and link them by URL instead.
- **Only scripts already on the page survive a rebuild.** A redesign cannot add a new
  analytics or widget script unless you allow its prefix. The same goes for `<base>` and
  `<meta http-equiv>` tags.
- **Dropped assets are detected by text.** A FULLHTML rebuild gets an old `<script src>` or
  font link re-added only if its URL appears nowhere in the new document, so a URL that is
  merely mentioned (in a comment, say) counts as present.
- **A model can still make a bad design choice** that is valid, safe CSS. pagepatch makes
  edits safe and checkable, not tasteful; keep a preview and an undo in your product.
- Tested on CPython 3.10, 3.13 and 3.14 on macOS.
- CSS rules added in a failed round of `propose` stay in the overrides block; a rule whose
  selector matched nothing is harmless, and `prune_overrides` can remove it later.

### Fill a page from data

Fill mode changes the bytes inside parts of a page that you marked, and nothing else. It never parses
the page into a tree and writes the tree back, so the doctype, the whitespace, void tags, inline
scripts and styles, and CRLF line endings all stay exactly as authored.

#### 1. Mark the page

```html
<ul data-pagepatch-region="menu">...</ul>                      <!-- the whole inner content is refilled -->
<b data-pagepatch-bind="loaf">Country sourdough</b>             <!-- one small element: its text and/or attributes -->
<a data-pagepatch-bind="order" href="/order">Order ahead</a>
<script type="application/json" data-pagepatch-region="stock">{}</script>   <!-- a data carrier -->
```

Add the markers by hand, or let `pagepatch mark` do it by CSS selector (below). Any element can carry a
marker; a marker found inside a comment, a `<script>` or a `<style>` is ignored. Both attributes are
`data-pagepatch-*` markers, so edit mode's page map shows them and warns when an edit drops one.
Use `--region-attr` / `--bind-attr` (or the `region_attr` / `bind_attr` arguments) to use your own names.

#### 2. Write the spec

```json
{
  "regions": {
    "menu": {
      "template": "<li>{name} <span>{price}</span></li>\n",
      "rows": [{"name": "Country sourdough", "price": "$9"}, {"name": "Rye & caraway", "price": "$10"}]
    },
    "stock": {"text": "{\"loaf\": 3}"}
  },
  "binds": {
    "loaf":  {"text": "Rye & caraway"},
    "order": {"attr": {"href": "/order?day=fri"}}
  }
}
```

| Entry | Meaning |
|---|---|
| `"regions": {id: "markup"}` or `{"html": "markup"}` | New inner content. Trusted markup, written as given (after the safety checks below). |
| `"regions": {id: {"text": "..."}}` | New inner content as plain text, HTML-escaped exactly once. Inside `<script>` and `<style>` it is written verbatim. |
| `"regions": {id: {"template": "...", "rows": [...], "empty": "..."}}` | One copy of `template` per row. `{name}` is escaped for text and for a quoted attribute value; a placeholder outside quotes inside a tag (`<a href={u}>`) is a spec error, so always quote attribute values; `{name\|url}` is escaped and becomes empty unless the URL is relative or http, https, mailto, tel or sms; `{name\|raw}` is inserted as is; `{{` and `}}` are literal braces. With no rows, `empty` is written (default: nothing). |
| `active_from` / `active_until` (ISO dates, inclusive) on a region object or on a row | With `--today`, a row outside its window is dropped, and a region outside it is cleared, so a dated notice removes itself. A missing or unreadable date counts as active. |
| `"binds": {id: {"text": "..."}}` | New text for every element carrying the marker (escaped once). |
| `"binds": {id: {"attr": {name: value}}}` | Set attributes on the start tag; only that tag is regenerated. Values are escaped and double-quoted. |

A key that is not in the spec, or whose value is `null`, leaves the page alone. An empty string clears.
Every element with a matching marker is filled. Typos in the spec's own keys are an error.

#### 3. Run it

```
pagepatch fill page.html spec.json -o out.html          # or -i to rewrite page.html, or no flag for stdout
pagepatch fill page.html spec.json -i --state sync.json --backup-dir backups
pagepatch fill page.html spec.json --dry-run --json     # just the report
```

| Option | Does |
|---|---|
| `-o FILE` / `-i` | Write the page to FILE, or over the input (only if it changed, and only if the file still holds what was read). Without either, the page goes to stdout. |
| `--state FILE` | Use with `-i`. Remember a fingerprint of each filled part. Next run, a part whose bytes no longer match what the last run wrote was edited by hand and is skipped (`SKIPPED`). The state is saved whenever the page is rewritten, also when another entry was refused. With `-o` it is a usage error (exit 2): the state describes the file you rewrite, and `-o` leaves the input as it was. |
| `--always-refresh ID` | Refill ID even if it was edited by hand. Binds are written `bind:ID`. Repeatable. |
| `--backup-dir DIR` | With `-i`, copy the previous page there first. |
| `--dry-run`, `--json` | Report only, or print the report as JSON instead of the page. |
| `--strict` | Also exit 1 for missing markers and skipped parts. |
| `--today YYYY-MM-DD\|now` | Apply the `active_from` / `active_until` windows as of that date. Without it nothing is filtered. (`pagepatch check` takes it too.) |
| `--sanitize` | Strip scripts and event handlers from region markup. |
| `--overrides-id ID` | The `<style id=ID>` block that a write may not shrink below half (default `pagepatch-overrides`). |

Exit codes: 0 done, 1 something was refused (or `--strict` and a part was missing or skipped; or the
final check failed), 2 a file or spec problem. The good entries are written even when another is refused.
A marker the spec names that is not on the page is always printed (`MISSING`), never ignored.

#### 4. What it refuses

* **Any write that changes a byte outside the filled parts.** After planning, `check_write` removes exactly
  the surfaces the spec writes (the inner content of its regions, the text of its binds, the named attributes
  of its binds) from the old and the new page and compares the rest. It is computed from the spec, not from
  the markers on the page: a bind that sets `href` still has its label inside the guard, and a marker the spec
  does not name is unmanaged. You can run it on its own on a page produced any other way:
  `pagepatch check before.html after.html spec.json`.
* **Region markup that is not self-contained**: it may not close an element it did not open, leave one open,
  or end inside a tag, comment or script. (`<li>` and `<p>` with optional end tags are fine.) `/>` counts only
  on void elements and inside `<svg>` or `<math>`, so `<div/>` is an open div and is refused; `<plaintext>`,
  `<xmp>`, `<noembed>` and `<noframes>` are refused. This is a balance check on the tags: it does not model
  every repair a browser makes to bad markup (stray text in a `<table>`, for example).
* **Content that would close a `<script>` or `<style>` region** (`</script>`, and `<!--` in a script). For
  data, use `js_json(value)`, which writes `<`, `>`, `&` and the JavaScript line separators as `\u` escapes.
* **Attribute writes that run code**: `on*` handlers, a `javascript:`, `data:` or `vbscript:` URL in
  `href`, `src`, `action` and the like (whitespace tricks such as `java<tab>script:` are seen through),
  attributes of `<script>`, `<iframe>`, `<object>`, `<embed>`, `<base>`, SVG animation elements
  (`<animate>`, `<set>`, `<animateMotion>`, `<animateTransform>`), `<meta http-equiv>` and stylesheet or
  preload `<link>`; adding `http-equiv` to a `<meta>` or a stylesheet or preload `rel` to a `<link>`; and
  the marker attributes themselves.
* **Nested regions** (the outer would swallow the inner), nested elements sharing a text bind, text into a
  void element such as `<br>`, and a text bind on the same element as a region being filled or around one
  (it would overwrite the region). A bind inside a region is allowed: the region is written first, then the
  bind, and the region's fingerprint is taken after both.
* A malformed page (an unclosed marked element, an unterminated comment): `RegionError`, exit 2.

#### From Python

```python
import pagepatch

html = pagepatch.read_page("page.html")            # bytes in, bytes out: CRLF files stay CRLF
state = pagepatch.load_state("sync.json")           # {} the first time
plan = pagepatch.fill(html, spec, prev_fingerprints=state, always_refresh=["stock"])

plan.changed, plan.skipped_edited, plan.missing, plan.refused, plan.ok, plan.noop
pagepatch.write_page("page.html", plan.html, expect_html=html, backup_dir="backups")
pagepatch.save_state("sync.json", {**state, **plan.fingerprints})
```

`fill` is `plan_fill` followed by `check_write`; it raises `SpecError` for a malformed spec and
`WriteRefused` if the result would change anything outside the filled parts. `write_page` replaces the file
in one step through a temporary file beside it, and refuses if the file no longer holds `expect_html`.
Lower level, all exported: `find_elements`, `set_inner_html`, `set_attr`, `region_fingerprint`,
`outside_hash`, `render_rows`, `js_json`, `url_is_safe`, `fragment_problem`, `check_template`.

#### Add the markers: `pagepatch mark`

```
$ cat targets.json
{"regions": {"menu": "ul.menu"}, "binds": {"loaf": "p.today b", "cta": {"select": "a.cta", "all": true}}}
$ pagepatch mark page.html targets.json -o marked.html
added   region:menu
added   bind:loaf
added   bind:cta
```

Each selector must match exactly one element (or use `"all": true` for every match). If any selector matches
nothing, matches too many, or targets an element already marked with a different id, no page is produced and
every problem is listed. Running it again on its own output changes nothing. Only the start tags of the chosen
elements change, by one added attribute each, and the result is re-checked.

#### Ship a new template without wiping the live page: `pagepatch merge`

A template in your repository and the copy that is live differ: a sync has refilled some regions and an editor
has changed some sections and the overrides stylesheet. Publishing the template verbatim wipes that.

```
pagepatch merge template.html live.html -o merged.html --region events --region banner --id nav --overrides-id site-overrides
```

The named regions, sections (by `id`) and the overrides `<style>` block keep the live copy's content inside the
template's markup. Each part must appear exactly once in each page (or in neither); anything else is refused
rather than guessed. Afterwards it lists every line of visible text a visitor could read on the live page that
the merged page no longer has, and refuses to write when there are any, unless you pass `--allow-loss` (for an
intentional redesign) or `--reset-overrides` (ship the template's own overrides block). In Python:
`merge_preserving(template, live, regions=[...], ids=[...])` returns the page plus `carried`, `absent`, `lost`.

#### See the markers: `pagepatch regions page.html [--json]`

Prints each marker with its tag, line number and size (regions also get their fingerprint).

#### Limits of fill mode

* **The scanner is a small tokenizer, not a full HTML parser.** It understands quoted attributes, comments,
  raw-text elements (`script`, `style`, `textarea`, `title`) and void or `/>` tags. A marked element is matched to
  its closing tag by counting same-name tags, so it needs an explicit closing tag; optional end tags are not
  inferred. A `<` that does not begin a well-formed tag is treated as text.
* **`mark` picks elements with BeautifulSoup's `html.parser` tree**, which does not infer optional end tags
  (`<li>`, `<p>` left open), so a selector that depends on that structure can differ from what a browser builds.
  Check the result with `pagepatch regions`.
* **Region markup is trusted** unless you pass `--sanitize`. The checks above keep its tags balanced and inside
  its region; they do not make hostile markup harmless (an `onclick` in region markup is written as given). Do not put untrusted HTML in a spec.
* **It is not a template language.** There are no conditions and no loops beyond `rows`. Compute the data first.
* **Hand-edit detection is a fingerprint, not a diff**: it tells you a part changed, not what changed.
* **`write_page` is not a lock.** It compares the file with what you read, then replaces it; a writer that slips in
  between those two steps is still lost. Run one sync job per page.
* **`merge` compares text nodes.** If the new template wraps the same sentence in different inline tags, those
  lines are reported as lost; treat the list as a backstop to read, not a proof of loss.
* **Speed**: every changed entry rescans the page. On a 430 KB page with 50 changed entries a fill took about
  three seconds; a typical page with a handful of entries takes milliseconds.
* Tested on CPython 3.10 and 3.14 on macOS.

### Running the tests

```
python -m pytest
```

The browser-verifier tests need `node` on the PATH (or in `/opt/homebrew/bin`,
`/usr/local/bin` or `/usr/bin`); they are skipped without it.

---

Docs mirror of the private repository Dominares-Tools/pagepatch; updated with each release.
