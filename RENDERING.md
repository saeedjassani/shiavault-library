# Rendering the library

The books are plain CommonMark with footnotes. This note records what the
Markdown in `books/` guarantees, and what a reader application (Shia Companion,
shiavault.com, or anything else) has to do to display it correctly.

## What the Markdown guarantees

Every chapter file is **pure CommonMark**, with two deliberate exceptions listed
under [Remaining HTML](#remaining-html). In particular:

* Arabic quotations are Markdown blockquotes (`> …`), never HTML.
* A quotation is one blockquote. It is never split across a Markdown quote and
  a second block.
* Footnote definitions are `[^n]: text`. Footnote references are `[^n]`.
* Headings are setext (`===` / `---`) or ATX (`##`).
* A run of dashes on its own line after a blank line is a horizontal rule; the
  same run directly under a line of text is a setext heading. Both forms occur.

`tools/normalize_markdown.py` enforces all of this and is idempotent. Run it
after importing or hand-editing a book:

```sh
python3 tools/normalize_markdown.py books          # fix in place
python3 tools/normalize_markdown.py --check books  # report only
python3 tools/verify_against_head.py               # prove no text was lost
```

## What the reader must do

### 1. Set `dir="auto"` on every rendered block

**This is the one requirement the Markdown cannot express itself.**

Arabic passages carry no direction marker — the text is its own signal. Every
block-level element the renderer emits (`p`, `blockquote`, `li`, `td`, heading)
needs `dir="auto"` so the browser or text engine picks direction from the first
strong character in the block.

```html
<blockquote dir="auto">إنَّمَا الأَعْمَالُ بِالنِّيَّاتِ.</blockquote>
```

In CSS, the equivalent is `unicode-bidi: plaintext` on those elements. In
Flutter, pass `textDirection: null` and let `Directionality` resolve per block,
or set `TextAlign.start` with the paragraph's detected direction.

Without this, Arabic renders left-aligned with punctuation stranded on the wrong
side. With it, the ~71,000 quotation blocks in the library lay out correctly
with no per-file markup.

### 2. Do not enable "hard line breaks"

Source lines are hard-wrapped at ~72 columns. This is normal CommonMark: a
single newline is a space, and only a blank line starts a new paragraph.

If the renderer is configured with `breaks: true` (marked, markdown-it) or
`softLineBreak: true`-style behaviour, every wrapped line becomes a `<br>` and
paragraphs turn into ragged one-line-per-clause blocks. **Leave that option
off.**

### 3. Support backslash escapes

The corpus contains ~66,000 escaped backticks (`` \` ``), used for the `ayn`
in transliteration (`` \`Ali ``, `` La\`na ``), plus escaped `_`, `*`, `#` and
`>`. These are correct CommonMark escapes and must be unescaped at render time.
A renderer that ignores them shows a stray backslash before the character; one
that also ignores the backtick's meaning can swallow a whole paragraph into a
code span.

### 4. Support footnotes, including book-level notes chapters

Footnotes use the standard `[^n]` / `[^n]:` extension (pandoc / markdown-it-
footnote / GFM-style). Two layouts occur:

* **Per-chapter** — definitions sit at the end of the same file. These resolve
  normally.
* **Per-book** — the chapter files carry only references, and one `Notes`
  chapter at the end of the book carries every definition. 858 files rely on
  this.

In the per-book case a reference has no definition in its own file. The reader
should resolve footnotes **across the book**, not just the current chapter, or
at minimum link an unresolved reference to the book's notes chapter rather than
rendering a dead `[^n]` marker.

### 5. Treat setext underlines as headings, not rules

`---` under a line of text is a level-2 heading. Roughly 45,000 headings in the
library use this form, against ~6,300 genuine horizontal rules (a run of dashes
after a blank line). A renderer that reads the underline as a rule loses the
entire heading hierarchy — and with it the table of contents and any
scroll-position or chapter-progress feature built on it.

## Remaining HTML

Two things are intentionally still HTML:

* **Tables** — 77 files. `<table>` markup with `colgroup`, alignment and cells
  that span lines. Markdown pipe tables cannot express all of it, so it is left
  as-is. Readers should pass it through, or fall back to rendering the cell
  text in document order.
* **Autolinks** — `<https://example.org>` and `<name@example.org>` are ordinary
  CommonMark autolinks, not raw HTML.

Nothing else in `books/` contains HTML tags.

## Known content issues not addressed here

These are transcription defects in the source texts, not rendering bugs. They
need a human reading the original book, so the normalizer leaves them alone:

* A handful of mojibake fragments left by the original conversion (for example
  a stray `A30B` where a Qur'anic verse separator belonged).
* Emphasis fragmented mid-phrase in a few books (`*al* *‑* *\`umr*`), which
  renders as several adjacent italic runs instead of one.
* Occasional missing spaces around emphasis (`are**مبني** in its entirety`).
