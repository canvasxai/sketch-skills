# Visual Identity — Org Context

Colors, typography, logo, and assets that define how the org looks on decks, docs, PDFs, and web pages. Consumer skills (docx, pdf, pptx, md-to-pdf, landing-page-copy) read this when producing branded output.

Blank fields are fine — skills fall back to neutral defaults when a field is empty.

<!--
AUTHORING RULES (see visual-identity/SKILL.md for full text):
- Replace the ENTIRE value; never leave hint text inside a field.
- Values are values. Annotations go in Notes columns or separate fields.
- File paths must be absolute LOCAL paths. Download remote URLs into assets/ first.
- Keep the canonical palette rows (Primary/Background/Text/Accent). Add brand-specific rows BELOW them.
- Extend sections by appending new fields; never rename existing sections or field labels.
-->

## Palette

Canonical rows (`Primary`, `Background`, `Text`, `Accent`) must stay — consumer skills look them up by name. Add brand-specific rows below the canonical four.

| Role | Hex | Notes |
|------|-----|-------|
| Primary | | |
| Background | | |
| Text | | |
| Accent | | |

## Typography

<!-- Font names as used in CSS / OOXML. Include a fallback stack for web rendering. -->

- **Heading font:** 
- **Body font:** 
- **Code / mono font:** 

### Typography notes

<!-- Casing rules, weight conventions, size scale, anything that isn't the font-family itself. -->

- **Heading casing:** 
- **Weight conventions:** 
- **Other notes:** 

## Logo

<!--
Logo values MUST be absolute local file paths — not URLs.
pptx and docx cannot embed images from HTTPS.
If the source is a URL, download it first (via the canvas skill or curl) into
~/.claude/skills/visual-identity/assets/<filename>, then record the local path.
Keep the original URL in an HTML comment for provenance if you want.
-->

- **Primary (color, on light bg):** 
- **Monochrome (for dark bg):** 
- **Icon only (square mark):** 
- **Tagline to pair with logo:** 

## Assets root

<!-- Optional local directory for images, icons, template files. If set, skills can reference files relative to this path. -->

- **Path:** 

## Brand stylesheet (for md-to-pdf and other HTML-rendered output)

<!--
Optional hand-crafted CSS with the brand's typography, spacing, and accent rules.
When set, md-to-pdf loads this alongside its bundled default.css.
pptx and docx ignore this — they read the token fields above directly.
Conventionally at ~/.claude/skills/visual-identity/assets/<brand>.css
-->

- **Path:** 

## Templates

<!--
Paths to brand-styled template files. When present, pptx/docx skills insert
content into the template's masters/layouts instead of building from blank.
All paths must be absolute LOCAL paths.
-->

- **Deck template (.pptx):** 
- **Doc template (.docx):** 
- **PDF cover template:** 

## Last verified

<!-- Stamp this whenever you change any field. Helps future sessions spot stale data. -->

- **Date:** 
