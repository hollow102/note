---
name: html-result-output
description: Use when writing an HTML result/deliverable file (design notes, research reports, task lists, summaries) that the user will open in the Agentic HTML Visual Editor VSCode extension. Covers the supported tag set (including collapsible details, strikethrough, GitHub-style alert blockquotes, and Mermaid diagram source blocks), the custom <comment> annotation tags, what the extension strips or forbids, and how to keep the markup minimal and semantic.
---

# Authoring HTML result files for Agentic HTML Visual Editor

This project is a VSCode extension that renders and edits `.html` files in a
WYSIWYG view. The HTML on disk is the single, portable source of truth shared
by the AI agent and the human. When you (the agent) produce an HTML deliverable
for the user, write it so it renders correctly in that view and stays minimal.

## Core principles

1. **Minimize output.** The whole point of the extension is to cut the context
   an agent emits. Prefer the smallest correct markup. Do not add framework
   wrappers, utility classes, `<div>` soup, inline scripts, or boilerplate the
   view does not need.
2. **Emit a full document with a `<body>`.** The renderer locates content by
   splitting around the `<body>` tag. Always output a complete skeleton:
   `<!DOCTYPE html>` → `<html>` → `<head>` (with `<meta charset>` and a
   `<title>`) → `<body>` … `</body>`. Content placed outside `<body>` is not
   rendered. Everything from the opening `<body …>` tag back to the doctype, and
   the closing `</body>…</html>`, is preserved verbatim — only body content is
   sanitized.
3. **Don't ship your own CSS framework.** The extension bundles a modern
   default stylesheet. Only use the `style` attribute for genuinely per-element
   intent (see *Styling* below). Inline `style` wins over the bundled CSS, so use
   it sparingly and deliberately.
4. **Write semantic HTML.** Use headings, paragraphs, lists, and tables for
   their meaning. The user reads and edits this; clean structure is the product.

## Supported tags

Stay inside this set — anything else may be stripped or render unstyled:

- **Inline:** `strong`, `em`, `code`, `s` (strikethrough), `a`, `span`
- **Block:** `h1`–`h6`, `p`, `blockquote`, `pre`, `hr`, `div`
- **Collapsible:** `details`, `summary` (see below)
- **Lists:** `ul`, `ol`, `li` (nesting allowed)
- **Tables:** `table`, `thead`, `tbody`, `tfoot`, `tr`, `th`, `td`, `colgroup`,
  `col` — with `colspan`, `rowspan`, `scope`
- **Media:** `img` (see below)
- **Diagrams:** `pre` carrying Mermaid source (see *Mermaid diagrams* below)
- **Custom:** `comment`, `comment-body`, `comment-reply` (see below)

Notes:

- **Prefer `strong`/`em` over `b`/`i`.** In the view, `<b>` is rewritten to
  `<strong>` and `<i>` to `<em>` on edit, so emit the semantic tag directly.
- **Code blocks:** wrap a `<code>` inside `<pre>`: `<pre><code>…</code></pre>`.
  `<pre>` and `<table>` subtrees are treated as opaque, so their internal
  whitespace is preserved exactly.
- **Mermaid diagrams:** emit the source as a `<pre>` block — the two supported
  forms and the rules that govern them are in *Mermaid diagrams* below.
- **Images:** use `img` with a `src` and an `alt`. `src` may be an `http(s)://`
  URL or a **relative path** (e.g. `./images/foo.png`) resolved from the HTML
  file's own directory. Optional `width`/`height` may be set via inline `style`.
- **Links:** in the view a link is followed with **Ctrl+click** (Cmd+click on
  macOS); a plain click places the caret so the link text stays editable. A
  relative link to a local file (e.g. `<a href="./notes.html">`) opens that file
  in a VSCode tab instead of navigating the view away, resolved within the
  document's workspace folder; other links keep their native behavior. Use
  relative hrefs to cross-link companion documents.

## Collapsible sections (`details` / `summary`)

Use `details` + `summary` for content the reader can expand or collapse. Put the
`<summary>` first as the clickable title; everything after it is the body. Add
the boolean `open` attribute to render the section expanded initially; omit it
for collapsed. The open/closed state is real and persisted back to the HTML.

```html
<details open>
  <summary>Implementation notes</summary>
  <p>Details that can be folded away.</p>
</details>
```

## GitHub-style alert blockquotes

Use a `blockquote` with `data-alert` when content needs an emphasized Note,
Tip, Important, Warning, or Caution presentation. One example per allowed value:

```html
<blockquote data-alert="note">Useful information the reader should notice.</blockquote>
<blockquote data-alert="tip">Advice for doing something better or more easily.</blockquote>
<blockquote data-alert="important">Key information the reader must not miss.</blockquote>
<blockquote data-alert="warning">Content that needs the reader's careful attention.</blockquote>
<blockquote data-alert="caution">Risks or negative outcomes of a certain action.</blockquote>
```

Allowed values are `note`, `tip`, `important`, `warning`, and `caution`. Keep
the label and icon **out** of the HTML — the WYSIWYG stylesheet supplies the
fixed English label, icon, and accent colour. A plain `<blockquote>` (no
`data-alert`) remains an ordinary quotation; an unrecognized `data-alert` value
also renders as an ordinary blockquote.

## Mermaid diagrams

Write a diagram as Mermaid **source** in either supported form. The view renders
it with its own bundled Mermaid runtime and preserves the form you chose:

```html
<pre class="mermaid">graph TD
  A[Draft] --> B[Review]
</pre>

<pre><code class="language-mermaid">sequenceDiagram
  Agent->>Human: deliverable.html
</code></pre>
```

Rules:

- **Never add a `<script>` or a CDN reference.** The runtime ships with the
  extension and loads on demand; a `<script>` is stripped on render anyway.
- **Emit source, never SVG.** The rendered diagram is presentation only — it is
  never written back to the file and never copied. Do not paste generated SVG
  into the document as a substitute for the source block.
- **Don't pin a theme** (`%%{init: {'theme': …}}%%`). The view renders in the
  active VSCode light/dark theme, and a pinned theme fights it.
- **Keep labels plain text.** Rendering runs with `securityLevel: 'strict'` and
  `htmlLabels: false`, so `click` directives are inert and HTML inside a label
  shows up as literal text.
- **Escape `<` and `&`** as `&lt;` / `&amp;` inside the source, as in any
  `<pre>` — otherwise the parser rewrites them and the saved file drifts.
- **Stay inside the runtime limits:** 50,000 characters of source and 500 edges
  per diagram. Split a bigger picture into several diagrams.
- Any diagram type the bundled Mermaid 11 supports (flowchart, sequence, class,
  state, ER, gantt, pie, mindmap, timeline, …) renders. Invalid syntax is not
  destructive: the block shows an error card and the source stays editable.
- The `<pre>` is opaque like a code block, so its indentation — which Mermaid is
  sensitive to — survives the round trip exactly as written.

## Styling

The extension bundles a modern default stylesheet, so unstyled semantic markup
already looks good. Reach for the `style` attribute only for genuine
per-element intent. The properties that carry real, user-meaningful meaning
(and survive the paste sanitizer) are:

- `color` and `background-color` — on any element.
- `text-align` — on block elements (`p`, `h1`–`h6`, `blockquote`, `pre`, `div`,
  `li`, `td`, `th`).
- `width` (`min-width`/`max-width`) — on table-sizing elements (`table`, `col`,
  `colgroup`, `th`, `td`).
- `width`/`height` — on `img`.

Avoid font, spacing, border, and layout styling; let the bundled CSS own the
look. Never add `class` for styling — there is no external stylesheet to match
it, and pasted `class` attributes are dropped. The only semantic class
exceptions are `mermaid` and `language-mermaid` in the diagram forms above.
Likewise never author a `data-ahve-*` attribute: those are the view's own
transient UI state, written and removed as you edit and stripped on save.

## IMPORTANT — Custom `<comment>` annotation tags

The extension's headline feature is an inline, review-style annotation that
lives **entirely in the HTML** so any reader — human or AI — can see the full
thread by inspecting the markup. Treat these tags as first-class and important.

### Structure

```html
<comment id="c-a1b2c3d4">target text
<comment-body contenteditable="false" data-author="ai" data-updated="2026-07-16T09:00:00Z">body text</comment-body>
<comment-reply contenteditable="false" data-author="ai" data-updated="2026-07-16T09:05:00Z">a reply</comment-reply>
</comment>
```

- `<comment id="…">` wraps the **inline run of target text** being annotated.
  In the view it renders as a highlighted, boxed run; clicking it opens a popup
  showing the body and replies. The box colour is keyed off the entry author, so
  human- and AI-authored comments are visually distinct.
- `<comment-body>` — **zero or one** per comment. Holds the main note.
- `<comment-reply>` — **zero or more** per comment, in document order. Each holds
  one reply.

### Document order

Emit the children in this canonical order and keep it: **target text first,
then `<comment-body>`, then `<comment-reply>` elements** (each reply in
chronological order).

### Author and timestamp metadata (required when you author)

Each `<comment-body>` and `<comment-reply>` records who wrote it and when:

- `data-author` — set to **`"ai"`** on every body/reply **you** write. (The
  human side uses `"human"`, possibly a named username in the future. `"ai"` is
  the fixed label for you, and the extension keys "is this the counterpart?"
  off exactly this value.)
- `data-updated` — an **ISO 8601** timestamp (e.g. `2026-07-16T09:00:00Z`).
  Use the current date/time when you create the entry.

The `<comment>` parent may carry a boolean `data-resolved` attribute marking the
thread as resolved. **Resolving is the human reviewer's action — do not add or
remove `data-resolved` yourself.**

### Rules to follow when emitting comments

- **`id` format:** `c-` followed by **8** lowercase alphanumeric characters
  (`[a-z0-9]`), e.g. `c-a1b2c3d4`. Each `id` must be **unique within the
  document**.
- **`comment` is inline** — place it inside a block (a `<p>`, `<li>`, `<td>`,
  etc.), wrapping only the phrase it annotates. Do not wrap whole blocks.
- **Don't put block elements inside a comment.** Keep the target text and the
  body/reply text as plain inline text; comments never nest.
- **Lock the children:** put `contenteditable="false"` on every
  `<comment-body>` and `<comment-reply>` so the user cannot accidentally type
  into them in the view. (The extension re-applies this, but emit it anyway.)
- **Stamp author + time:** every body/reply you write gets `data-author="ai"`
  and a current `data-updated` ISO 8601 timestamp (see above).
- **Append only — never modify existing comments.** When revising a document
  that already has comments, you may **add** new `<comment>` elements and **add**
  new `<comment-reply>` entries to existing comments. Do **not** edit, reword,
  delete, re-author, or re-timestamp any existing `<comment>`, `<comment-body>`,
  or `<comment-reply>` — especially ones authored by the human (`data-author`
  other than `"ai"`). Leave their text, `data-author`, and `data-updated`
  untouched. To respond to a human note, add a new `<comment-reply>`.

### When to use a comment

The body and replies are hidden from the document flow visually but remain in
the HTML source, so they are the intended channel for agent↔human review notes
inside the deliverable. Use a `<comment>` to flag something the user should
decide, verify, or be aware of without disrupting the readable flow of the
document — e.g. "I assumed X here", "needs a source", "two options, picked the
first". This is preferable to inlining meta-notes into the prose.

Note: comments are **private to the editor**. When the user copies or exports
HTML, the `<comment-body>`/`<comment-reply>` are dropped and the `<comment>`
wrapper is unwrapped, leaving only the commented-on text. Do not rely on comment
content surviving a copy out of the editor.

## What the extension forbids or strips

Do not emit these — they are removed on render (and some are security-sensitive):

- **Never executed / dropped tags:** `script`, `iframe`, `object`, `embed`,
  `frame`, `frameset`, `noscript`, `link`, `style`, `base`, `meta` (inside
  `<body>`).
- **Event handlers:** any `on*` attribute (`onclick`, `onload`, …) is stripped.
- **Unsafe URLs:** values in `href`, `src`, `xlink:href`, `srcset`, `action`,
  or `formaction` starting with `javascript:`, `vbscript:`, or `data:text/html`
  are dropped.
- **No external CSS/JS:** external stylesheets and scripts are not loaded.
  Mermaid support is provided by the extension's own bundled runtime.
- **Forms render but don't submit:** a `<form>` shows but submission is disabled.
- **Comment annotations are stripped from exported HTML** — see the note above.
- **Mermaid previews are reduced back to their source block** on save and on
  copy, so the file never accumulates rendered output.

## Quick checklist before delivering

- [ ] Full document with `<head>` (charset + title) and a real `<body>`.
- [ ] Only supported tags; `strong`/`em` (not `b`/`i`); no
      `script`/`iframe`/`style`/`on*`/unsafe URLs.
- [ ] Inline `style` limited to meaningful properties; no bespoke CSS framework
      and no styling `class` attributes (apart from the two Mermaid tokens).
- [ ] Alerts use `<blockquote data-alert="…">` with no label/icon markup;
      `details` has `<summary>` first and `open` only when it should start open.
- [ ] Mermaid diagrams are source blocks in one of the two supported forms —
      no `<script>`, no CDN, no generated SVG, no pinned theme.
- [ ] Any `<comment>` you add has a unique `c-` + 8-char id, canonical child
      order (target → body → replies), and `contenteditable="false"` on its
      body/replies.
- [ ] Every body/reply you author has `data-author="ai"` and a current
      `data-updated` (ISO 8601). You did not touch any existing comment.
- [ ] Markup is as small as it can be while staying semantic and readable.

````
