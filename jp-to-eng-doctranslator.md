---
name: jp-doc-translator
description: "Translate Japanese-language Word (.docx/.dotx), Excel (.xlsx/.xlsm), or PDF (.pdf) documents into a single readable English Markdown file, preserving headings, tables, and images. Use whenever a user uploads or references a Japanese (or mixed-language) document and wants it translated, made readable, or converted to Markdown — e.g. 'translate this file', 'what does this Japanese doc say', 'convert this to English markdown'. Also use as the first step if the user wants the content in some other English format (report, summary), since the Markdown translation is the natural intermediate artifact. Do NOT use for documents already in English, for translating short pasted snippets (translate inline instead), or when the final deliverable must itself be an English .docx/.xlsx/.pdf (use the docx/xlsx/pdf skills for that, optionally after this skill produces clean English source content)."
---

# Japanese Document → English Markdown Translator

Takes a Word, Excel, or PDF file that contains Japanese text (possibly mixed
with English or other languages) and produces a single, clean Markdown file
that reads naturally in English, with the original structure — headings,
tables, images — preserved.

## Why this is two separate steps

Getting from "Japanese file" to "readable English markdown" requires two very
different kinds of work, and keeping them separate is what makes the output
good:

1. **Structural extraction** (mechanical, deterministic) — pulling headings,
   paragraphs, tables, and images out of a binary file format. This is what
   the bundled scripts do. Don't try to hand-roll this per file type; the
   scripts already handle the format-specific mess (docx XML, xlsx sheets,
   PDF text/table/image layout).
2. **Translation** (judgment-based) — turning the extracted Japanese into
   natural, accurate English while keeping non-Japanese content (numbers,
   proper nouns already in English, code, URLs) untouched. This is your job,
   not a script's — translation quality depends on context, and you're much
   better at reading intent (e.g. Japanese business idioms, honorifics that
   don't need literal rendering) than any regex or library would be.

Run extraction first to get a structured but still-Japanese `raw.md`, then
translate it yourself into the final output.

## Workflow

### 1. Set up a working directory and identify the file type

```bash
mkdir -p /tmp/jpdoc-<slug>
```

Look at the file extension (and if ambiguous, `file <input>`) to decide which
extractor to run:

| Extension | Script |
|---|---|
| `.docx`, `.dotx` | `scripts/extract_docx.py` |
| `.xlsx`, `.xlsm` | `scripts/extract_xlsx.py` |
| `.pdf` | `scripts/extract_pdf.py` |

Legacy formats (`.doc`, `.xls`) need conversion first — the docx/xlsx skills
both bundle a LibreOffice conversion script
(`scripts/office/soffice.py --headless --convert-to docx/xlsx`); use that,
then extract the converted file.

### 2. Run the extractor

```bash
python3 scripts/extract_docx.py <input> /tmp/jpdoc-<slug>
# or extract_xlsx.py / extract_pdf.py, same signature
```

This writes `/tmp/jpdoc-<slug>/raw.md` (structure preserved, text still in
whatever language it was in) and `/tmp/jpdoc-<slug>/media/` (extracted
images, if any).

**Read `raw.md` fully before translating anything** — skim it once to get a
sense of document length and structure so you can decide whether to
translate in one pass or in chunks (see "Long documents" below).

**If a page is flagged `[NEEDS_OCR: ...]`** (PDF extractor only — the page is
a scanned image with no text layer), fall back to reading that page visually:
render it with `pdftoppm -jpeg -r 150 <input.pdf> page -f <N> -l <N>` and view
the resulting image directly, then transcribe and translate what you see in
its place in the markdown. Don't silently drop the page.

### 3. Translate into the final Markdown

Rewrite `raw.md` into `translated.md`, sentence by sentence and section by
section, applying these rules:

- **Translate all Japanese to natural English.** Prioritize meaning and
  readability over literal word-for-word translation — this should read like
  it was written in English, not like a translation.
- **Leave already-English (or other non-Japanese) content untouched.**
  Mixed-language documents are common (e.g. an English product name inside a
  Japanese sentence); don't touch text that isn't Japanese.
- **Preserve structure exactly**: heading levels, table shape (same rows and
  columns, just translated cell contents), list nesting, and the position of
  image references (`![](media/...)`). Someone should be able to skim the
  translated doc and match it 1:1 against the original's layout.
  - Sheet/section headings from Excel (`## Sheet Name`) get translated too if
    the sheet name itself is Japanese.
  - Table headers and cell values both get translated, but keep numbers,
    dates, and formula-derived values exactly as extracted.
- **Keep proper nouns recognizable.** Company and product names usually stay
  as-is or in their standard English form (e.g. 株式会社トヨタ →
  Toyota Motor Corporation) rather than a literal gloss. Person names get
  standard romanization (Hepburn) unless a common English form already
  exists.
- **Don't editorialize or summarize.** Translate what's there; don't skip
  sections because they seem repetitive or add commentary about the
  document. If something is genuinely illegible or ambiguous, leave a short
  inline note like `[translation uncertain: ...]` rather than guessing
  silently.

#### Long documents

For large `raw.md` files, translate in logical chunks (e.g. one heading
section, or one sheet, or every ~5 pages at a time) rather than trying to
hold the whole thing in your head at once — but keep a mental glossary of any
recurring proper nouns or domain terms so translations of the same term stay
consistent across chunks. If you notice you translated a term differently
earlier in the document, go back and fix it for consistency before finishing.

### 4. Finalize and save the output

- Copy the `media/` folder alongside the final markdown file (image
  references in the markdown are relative paths like `media/p1_i1.png`, so
  this must sit next to `translated.md` for images to render).
- Save the final file to `/mnt/user-data/outputs/<original-name>.md` (or a
  sensible name derived from the source file), with `media/` copied next to
  it if it's non-empty.
- Present the output file to the user with `present_files`.

## Common pitfalls

- **Don't skip the extraction scripts and try to read+translate a binary
  file directly** — you can't parse `.docx`/`.xlsx` XML or PDF byte streams
  reliably by eye; always go through the extractor first.
- **Don't translate table structure into prose.** A table should stay a
  table in the output, even if that means shorter, choppier English in each
  cell — reflowing it into paragraphs loses information (which value went
  with which row/column).
- **Don't drop images.** If `media/` has files, every one of them should be
  referenced somewhere in `translated.md`. If an image's original position
  in the document isn't recoverable (rare, mostly a PDF edge case), place it
  at the end of the section it was extracted from and note that its exact
  position is approximate.
- **Watch for vertical text and furigana in PDFs** — pdfplumber reads left
  to right, top to bottom, so vertically-set Japanese text or furigana
  (small reading-aid characters above kanji) can extract out of order,
  landing on their own interleaved lines rather than inline with the kanji
  they annotate. Usually you can still read straight through this — the
  furigana lines are short phonetic fragments that are easy to recognize
  and skip over mentally, and the real sentence is still there underneath.
  Only fall back to re-rendering the page as an image and reading it
  visually if the jumbling is bad enough that you're genuinely unsure what
  the underlying text says — don't do it as a default first step.
- **Watch for mojibake from broken font encoding.** Some PDFs (often ones
  produced by certain web-to-PDF renderers) embed fonts without a proper
  ToUnicode map, so `extract_pdf.py` pulls out nonsense characters instead
  of Japanese text — it'll look like random accented Latin/symbol garbage,
  not Japanese at all. If you see that, don't try to translate it: re-render
  the affected page as an image (`pdftoppm -jpeg -r 150 <input.pdf> page -f
  <N> -l <N>`) and read/translate it visually instead, same as the
  `NEEDS_OCR` fallback.
- **Table detection on PDFs is best-effort.** `find_tables()` relies on
  visible grid lines or consistent whitespace alignment, so borderless or
  loosely-aligned tables may not be detected as tables at all — their
  content still comes through in the plain page text, just without table
  structure. If a chunk of `raw.md` clearly reads like tabular data (short
  label/value pairs stacked line by line) even though it isn't in `| ... |`
  form, reconstruct it as a proper markdown table yourself when writing
  `translated.md`.
