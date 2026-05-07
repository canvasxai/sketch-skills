---
name: docx
description: >
  Create, read, and edit Microsoft Word documents (`.docx` files). Use this
  skill any time the user mentions "Word doc", "document", ".docx", "write up",
  "memo", "letter", "report", "brief", "contract draft", or attaches a `.docx`
  file. Triggers: "write me a memo about…", "draft a one-pager on…", "turn
  this into a Word doc", "read this contract and flag anything weird", "update
  section 3 of this doc", "format this as a proper report with headings and a
  table of contents", "summarise this .docx". Use even when the user only says
  "write up" or "document" — they usually mean a Word doc for sharing. Do NOT
  use for Google Docs (different format, needs the Google Drive integration),
  PDFs (use `pdf` skill), or slide decks (use `pptx` skill).
category: productivity
---

# DOCX

## What this skill does

Creates, reads, and edits Word documents. Handles headings, paragraphs, tables, images, tables of contents, page numbers, headers/footers, tracked changes, and comments. Uses `python-docx` as the default library.

## How files flow in Sketch

- **Incoming**: attached `.docx` files land in `attachments/<filename>.docx`. Read from there.
- **Outgoing**: write produced docs to `output/<slug>.docx` and deliver via `SendFileToChat`.
- **Do not overwrite** the original attachment — always save the edited version to a new filename (`doc-updated.docx`, `doc-v2.docx`).

## Brand context

At the start of every session, read `~/.claude/skills/visual-identity/SKILL-CONTEXT.md` if it exists. When **creating** a new document from scratch (memo, report, brief, letter), apply whatever's populated: brand palette for accent colors (headings, table headers, callouts), typography stack for heading and body fonts, and logo on the cover/header if a path is set.

### Templates (preferred path for branded docs)

If `visual-identity/SKILL-CONTEXT.md` has a **Doc template (.docx)** path set, open that template and insert content using its named styles — not a blank Document:

```python
from docx import Document
doc = Document("<doc template path from visual-identity>")
# Use the template's built-in heading styles so branded typography applies
doc.add_heading("Section", level=1)
doc.add_paragraph("Body...", style="Body Text")
```

Rules:
- Use the template's **named styles** (`"Heading 1"`, `"Heading 2"`, `"Body Text"`, etc.). Don't hand-roll fonts on individual runs — you'll undo the template.
- If the template includes a cover-page section, keep it and just fill its placeholders.
- Don't overwrite the template's header/footer; if the user wants different headers, they should update the template.

### If visual-identity is empty or missing — ASK before building

Do NOT silently default. Before producing the doc, ask:

> "I don't have your visual identity set up yet. Three options — pick one:
> 1. **Bootstrap it now** — I'll grab brand cues from your website (invokes the `visual-identity` skill).
> 2. **Use a template** — if you have a branded `.docx` template, upload it or point me at a path.
> 3. **Neutral defaults** — I'll use system fonts and a neutral palette; you can re-style later."

Apply the user's choice before proceeding. After delivery, offer to re-style with a template if they picked option 3.

### Existing docs

When **reading or editing** an existing document, don't restyle it — preserve the author's formatting. Only apply brand styling to content you're generating fresh.

Company identity (name, URL) comes from ambient memory / CLAUDE.md, or from `~/.claude/skills/identity/SKILL-CONTEXT.md` if set up.

## Creating a new document

```python
from docx import Document
from docx.shared import Pt, Inches, RGBColor
from docx.enum.text import WD_ALIGN_PARAGRAPH

doc = Document()

# Title (styled as Title)
doc.add_heading("Quarterly Review — Q2 2026", level=0)

# Heading
doc.add_heading("Executive summary", level=1)
doc.add_paragraph(
    "Revenue grew 34% QoQ, driven by enterprise expansion and improved "
    "retention. The top risk is concentration — three customers account for "
    "46% of ARR."
)

# Bulleted list
doc.add_heading("Highlights", level=1)
for point in ["34% QoQ revenue growth",
              "NRR climbed to 128%",
              "Two new enterprise logos"]:
    doc.add_paragraph(point, style="List Bullet")

# Numbered list
for step in ["Identify top 3 concentration risks",
             "Draft retention offer for at-risk accounts",
             "Review with GTM lead"]:
    doc.add_paragraph(step, style="List Number")

doc.save("output/q2-review.docx")
```

### Headings, styles, and structure

Use real heading styles (`level=1`, `level=2`, etc.) rather than bold-large text — this makes the doc navigable, generates a proper TOC, and renders correctly in Word's navigation pane.

### Tables

```python
table = doc.add_table(rows=1, cols=3)
table.style = "Light Grid Accent 1"
header = table.rows[0].cells
header[0].text = "Metric"
header[1].text = "Q1"
header[2].text = "Q2"

for metric, q1, q2 in [("Revenue", "$1.2M", "$1.6M"),
                       ("NRR", "118%", "128%")]:
    row = table.add_row().cells
    row[0].text = metric
    row[1].text = q1
    row[2].text = q2
```

### Images

```python
doc.add_picture("output/chart.png", width=Inches(5.5))
```

If the user wants charts, generate with matplotlib and insert as images. For native editable charts, a `.docx` can embed them but it's fiddly — usually images are fine for reports.

### Table of contents

Inserting a real auto-updating TOC requires writing raw XML (python-docx doesn't have a direct helper). Pattern:

```python
from docx.oxml.ns import qn
from docx.oxml import OxmlElement

def add_toc(doc):
    paragraph = doc.add_paragraph()
    run = paragraph.add_run()
    fldChar_begin = OxmlElement("w:fldChar")
    fldChar_begin.set(qn("w:fldCharType"), "begin")
    instrText = OxmlElement("w:instrText")
    instrText.set(qn("xml:space"), "preserve")
    instrText.text = 'TOC \\o "1-3" \\h \\z \\u'
    fldChar_sep = OxmlElement("w:fldChar")
    fldChar_sep.set(qn("w:fldCharType"), "separate")
    fldChar_end = OxmlElement("w:fldChar")
    fldChar_end.set(qn("w:fldCharType"), "end")
    for el in [fldChar_begin, instrText, fldChar_sep, fldChar_end]:
        run._r.append(el)
```

When the user opens the doc in Word, they'll see a "Right-click → Update Field" hint. Mention this in the chat reply: "The table of contents will populate when you open it in Word — right-click it and hit 'Update Field' the first time."

### Page numbers, headers, footers

```python
section = doc.sections[0]
footer = section.footer
p = footer.paragraphs[0]
p.alignment = WD_ALIGN_PARAGRAPH.CENTER
# For dynamic page numbers, insert a PAGE field via XML (similar pattern to TOC)
```

## Reading / summarising a document

```python
from docx import Document

doc = Document("attachments/contract.docx")
for para in doc.paragraphs:
    if para.style.name.startswith("Heading"):
        print(f"\n## {para.text}")
    else:
        print(para.text)

for table in doc.tables:
    for row in table.rows:
        print(" | ".join(cell.text for cell in row.cells))
```

When summarising in chat, lead with the structure (heading outline), then the substance. This is how people actually navigate docs mentally.

## Editing an existing document

**Find-and-replace pattern** — preserves formatting:

```python
for para in doc.paragraphs:
    for run in para.runs:
        if "OLD_TERM" in run.text:
            run.text = run.text.replace("OLD_TERM", "NEW_TERM")
```

Note: text can be split across multiple `run` objects within a paragraph, so naive find-and-replace on `para.text` loses formatting. If the search term crosses runs, rewrite the paragraph's runs as a single run with the new text.

**Inserting a new section** — python-docx doesn't have a "insert after paragraph X" helper. Build the doc by copying paragraphs up to the insertion point, adding new content, then copying the rest.

## Tracked changes and comments

python-docx has limited native support here — reading works; writing requires XML manipulation. If the user asks for tracked changes or comments on a large doc, do your best with XML insertion and flag the limitation in chat if something can't be done.

## Reply format

- **After building**: one-sentence summary of what's inside + `SendFileToChat`. Example: "5-page Q2 review with TOC, exec summary, three data tables, and recommendations."
- **After reading**: structured outline (headings) + key findings, not a full transcript.
- **After editing**: brief diff summary ("Updated section 2.3 pricing, added a new 'Risks' section between 4 and 5") + updated file.

## Error handling

- **Password-protected `.docx`** → ask the user for the password; python-docx can't read protected files directly.
- **Mixed old `.doc` format** → not supported by `python-docx`. Tell the user briefly and ask them to re-save as `.docx` from Word.
- **Very large docs** (100+ pages) → reading still works but can be slow. Warn the user and offer a section-by-section approach.
- **Never mention** `python-docx` or library internals to the user. Describe actions in plain terms ("I'll pull the headings and the numbers out of this doc…").
