---
name: pdf
description: >
  Manipulate existing PDF files — read, extract, summarise, OCR scanned PDFs,
  fill forms, merge, split, rotate, watermark-overlay. Use this skill whenever
  the user sends a PDF and asks about it. Triggers: "what's in this PDF",
  "extract the text/table from this PDF", "summarise this PDF", "fill out this
  form", "merge these PDFs", "split this PDF", "rotate these pages", "add a
  watermark", "OCR this scanned PDF", "is this PDF encrypted". Also use when
  the user attaches a `.pdf` file with any instruction — read it first before
  answering. Do NOT use this skill for CREATING PDFs from text content
  ("make a PDF of…", "turn this into a PDF", "export as PDF") — use the
  `md-to-pdf` skill instead, which produces much better typography. Do not
  mention Python libraries or internal tooling to the user; just do the work.
category: productivity
---

# PDF

> **For creating PDFs from content, use the `md-to-pdf` skill.** This skill handles PDFs that already exist.

## What this skill does

Reads and manipulates existing PDFs: text and table extraction, form filling, merging, splitting, rotation, watermark overlays, OCR on scans, and decryption. Works on attached PDFs (downloaded to `attachments/` in the workspace) and on PDFs produced elsewhere in the workspace.

## How files flow in Sketch

- **Incoming**: when the user attaches a PDF, it's already saved to the workspace under `attachments/<filename>.pdf`. Read directly from there — don't ask the user to upload again.
- **Outgoing**: write any produced PDF to the workspace root (or a sensible subfolder like `output/`) and deliver with `SendFileToChat` passing the absolute path.
- **Never** paste large extracted text blocks into chat. Summarise, or save the extraction to a file and send it.

## Brand context

Most operations here are on existing PDFs — don't restyle them, preserve the original. Brand context only matters for the narrow case of **generating a watermark PDF to overlay** (e.g. "DRAFT" or "CONFIDENTIAL" in the company colors). For that one case, read `~/.claude/skills/visual-identity/SKILL-CONTEXT.md` for palette/typography; if empty, use neutral defaults silently — not worth asking for a watermark.

For full PDF creation (reports, one-pagers, cover sheets), hand off to the `md-to-pdf` skill — don't build it here.

## Reading a PDF

**Default approach — `pypdf` for most work.**

```python
from pypdf import PdfReader

reader = PdfReader("attachments/contract.pdf")
print(f"Pages: {len(reader.pages)}")
text = "\n".join(page.extract_text() or "" for page in reader.pages)
```

**For tables or complex layouts — `pdfplumber`.**

```python
import pdfplumber

with pdfplumber.open("attachments/report.pdf") as pdf:
    for page in pdf.pages:
        for table in page.extract_tables():
            # each table is a list of rows
            ...
```

**For scanned PDFs (image-only, no text layer) — OCR with `pytesseract`.**

First check whether the PDF has a text layer: if `page.extract_text()` returns empty or near-empty across all pages, treat it as scanned and fall back to OCR.

```python
import pytesseract
from pdf2image import convert_from_path

pages = convert_from_path("attachments/scan.pdf", dpi=200)
text = "\n\n".join(pytesseract.image_to_string(img) for img in pages)
```

## Filling out a PDF form

**Use `pypdf` to detect fields first, then fill.**

```python
from pypdf import PdfReader, PdfWriter

reader = PdfReader("attachments/form.pdf")
fields = reader.get_fields()  # inspect field names

writer = PdfWriter(clone_from=reader)
writer.update_page_form_field_values(
    writer.pages[0],
    {"full_name": "Alex Smith", "email": "alex@example.com"},
)

with open("output/form-filled.pdf", "wb") as f:
    writer.write(f)
```

If the user sends a form and asks to fill it without specifying values, inspect `get_fields()` and ask them for the values in one concise message. Don't guess.

## Merging, splitting, rotating

```python
from pypdf import PdfWriter, PdfReader

# Merge
writer = PdfWriter()
for path in ["a.pdf", "b.pdf", "c.pdf"]:
    writer.append(path)
with open("output/merged.pdf", "wb") as f:
    writer.write(f)

# Split page range
reader = PdfReader("big.pdf")
writer = PdfWriter()
for i in range(4, 10):  # pages 5–10
    writer.add_page(reader.pages[i])
with open("output/pages-5-10.pdf", "wb") as f:
    writer.write(f)

# Rotate
reader = PdfReader("input.pdf")
writer = PdfWriter()
for page in reader.pages:
    page.rotate(90)
    writer.add_page(page)
```

## Creating a PDF from scratch

**Don't do it here.** For PDFs produced from text content (reports, one-pagers, handouts), use the `md-to-pdf` skill — better typography than any Python approach and matches how models produce content. If the user lands on this skill but actually wants to create, suggest the handoff:

> "For a clean PDF from that content, I'll switch to `md-to-pdf` — it produces much better typography."

## Watermarking

Overlay one PDF on another page-by-page using `pypdf`'s `merge_page`.

```python
from pypdf import PdfReader, PdfWriter

content = PdfReader("input.pdf")
watermark = PdfReader("watermark.pdf").pages[0]

writer = PdfWriter()
for page in content.pages:
    page.merge_page(watermark)
    writer.add_page(page)
with open("output/watermarked.pdf", "wb") as f:
    writer.write(f)
```

## Reply format

- **Extraction/summary** → reply in chat with the key findings. If the extraction is long (full document transcript, dozens of tables), save to a `.md` or `.txt` file and deliver via `SendFileToChat`.
- **Produced PDFs** → write to workspace, then `SendFileToChat`. In the chat reply, describe in one line what's inside ("2-page filled form with your details, ready to sign").
- **Never** paste raw extracted text dumps in chat — always summarise or offload to a file.

## Error handling

- **Encrypted PDF** (`PdfReader.is_encrypted == True`) → ask the user for the password before trying to extract. Do not guess.
- **Empty text extraction** → fall through to OCR (see above).
- **Library missing in sandbox** → tell the user briefly that the specific operation isn't available here, and offer a workaround (e.g., "I can pull out the text but table extraction isn't available — want me to try the text approach?").
- **Never mention** "pypdf", "reportlab", "OCR library", etc. to the user. Describe actions in plain language ("I'll pull the tables out of this report…").
