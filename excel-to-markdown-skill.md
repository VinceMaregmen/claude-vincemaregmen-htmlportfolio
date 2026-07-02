---
name: excel-to-markdown
description: "Converts Excel spreadsheets (.xlsx, .xlsm) into a Markdown (.md) file, with one section per sheet. Use this skill whenever the user wants to turn an Excel file into Markdown, a .md file, a text/document format, or wants spreadsheet data readable in a markdown viewer, README, wiki, or documentation site. Trigger this any time the user references an Excel/spreadsheet file by name or path and asks to convert, export, document, or turn it into Markdown - even if they don't say the word 'markdown' explicitly, e.g. 'can you make this xlsx readable in my docs' or 'dump this spreadsheet into a text file I can paste into Notion'. Preserves merged cells, formulas (as calculated values), and basic bold/italic styling as closely as Markdown/HTML allows. Do NOT use this skill when the desired output is itself a spreadsheet file (.xlsx/.csv) - that's the xlsx skill instead."
license: Complete terms in LICENSE.txt
---

# Excel to Markdown

## Overview

Converts an Excel workbook into a single Markdown file. Every sheet in the workbook gets its own `## Sheet: <name>` section, so multi-sheet workbooks stay organized and navigable in the output.

## Why this matters

A naive Excel-to-Markdown conversion just dumps values into a plain grid and loses everything else: merged headers collapse or get duplicated confusingly, formulas turn into blank cells or raw `=SUM(...)` clutter, and bold/italic emphasis disappears. This skill preserves as much of the original structure as Markdown (and embedded HTML, where necessary) allows, so the output still looks and reads like the original spreadsheet.

## Workflow

1. **Locate the input file.** Confirm the path to the `.xlsx`/`.xlsm` file (check `/mnt/user-data/uploads/` if the user uploaded it).
2. **Run the conversion script:**
   ```bash
   python /mnt/skills/.../excel-to-markdown/scripts/xlsx_to_md.py <input.xlsx> <output.md>
   ```
   (Adjust the skill path to wherever this skill is actually mounted.) If no output path is given, it defaults to the input filename with a `.md` extension.
3. **Sanity-check the result** by viewing the generated `.md` file before handing it off - especially sheets with merged cells or many formulas, since those are the trickiest to render correctly.
4. **Save the output** to `/mnt/user-data/outputs/` and present it to the user with `present_files`.

## How the conversion works

- **Multiple sheets**: each sheet becomes its own `## Sheet: <name>` section within one `.md` file, in the order the sheets appear in the workbook.
- **Merged cells**: Markdown's pipe-table syntax has no concept of a merged cell (no rowspan/colspan). For any sheet that contains merges, the script renders that sheet as a plain HTML `<table>` with `rowspan`/`colspan` attributes instead of a Markdown pipe table. This is still valid inside a `.md` file - GitHub, VS Code, Obsidian, and most renderers display embedded HTML tables correctly - and it's the only way to represent a merge without silently duplicating or dropping data. Sheets with no merges use normal Markdown pipe tables, which are cleaner and more portable.
- **Formulas**: the script reads the cached calculated value (what Excel/LibreOffice last computed and stored in the file) rather than the raw formula, since that's what a reader actually wants to see. If a cell has a formula but no cached value (rare - usually means the file was generated programmatically and never opened in a spreadsheet app), the script falls back to showing the formula text itself, clearly flagged as unevaluated, rather than silently leaving it blank.
- **Styling**: bold and italic fonts are mapped to Markdown's `**bold**` / `*italic*` emphasis (or the HTML equivalents inside merged-cell tables). Other formatting (colors, borders, fills, column widths) doesn't have a meaningful Markdown equivalent and is intentionally dropped rather than approximated badly.

## Edge cases to watch for

- **Very wide sheets** (many columns) can produce unwieldy tables. If a sheet has an impractical number of columns, mention this to the user rather than silently producing a huge table - they may want it split or transposed.
- **Empty sheets** render as `*(empty sheet)*` rather than an empty table.
- **Numbers stored as whole floats** (e.g. `5.0` from a formula) are cleaned up to display as `5` rather than `5.0`, since that matches what a user would actually see in Excel.
- **Pipe characters and newlines inside cell values** are escaped/converted so they don't break table formatting.
