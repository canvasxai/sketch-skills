---
name: md-to-pdf
description: >
  Create a PDF from text content. Convert markdown text or files into a
  nicely-styled PDF using the md-to-pdf npm package via npx. Use this skill
  whenever the user wants to PRODUCE a new PDF from text — "make a pdf",
  "convert to pdf", "export as pdf", "save as pdf", "turn this into a pdf",
  "/pdf", pastes markdown and asks for a printable/shareable document, or
  wants a PDF report, handout, or document from text content they've provided
  (even if they don't say "markdown" explicitly). This is the preferred path
  for any PDF-from-scratch generation because models produce better-structured
  content as markdown than as PDF-layout code. Do NOT use this skill for
  manipulating existing PDFs (extraction, OCR, merging, splitting, rotating,
  form-filling, watermarking a document you already have) — use the `pdf`
  skill for those.
---

# md-to-pdf

Converts markdown (inline text or a file) into a PDF using `npx md-to-pdf`. Output lands in the current working directory by default, styled with a bundled stylesheet tuned for readability.

## Why this is the create-PDF path

This skill is the canonical path for **creating** PDFs from content, not the `pdf` skill. Reason: models (including you) produce substantially better-structured content as markdown than as PDF-layout code — the abstraction matches how you think, so the output is cleaner, headings are sensible, tables render right, code blocks stay intact. The actual typography and layout are handled downstream by Chromium via `md-to-pdf`, which is far better than hand-rolling `reportlab` or `weasyprint` calls.

**Pattern to use across skills:**
- Agent writes content in markdown (what you're good at).
- md-to-pdf converts to styled PDF with CSS (what Chromium is good at).

The `pdf` skill is for **manipulating existing PDFs** (extract, OCR, merge, split, form-fill, watermark-overlay). When the user says "make a PDF", "export as PDF", "turn this into a PDF", "save this as PDF" — that's this skill, not `pdf`.

## When to use

- User provides markdown content in the conversation and asks for a PDF
- User points at a `.md` or `.txt` file and asks to convert it
- User asks to "export", "save", "turn into", or "make" a PDF from text
- Another skill has produced text content (a research brief, a landing page draft, an ICP summary) and the user wants it as a shareable PDF — write a markdown file from that content first, then run this skill

If the user wants something other than text-to-PDF (e.g. HTML-to-PDF with a custom layout, HTML form rendering, web page archiving), this skill is the wrong fit — use Puppeteer directly or suggest `wkhtmltopdf`.

## Brand context

At the start of every session, read `~/.claude/skills/visual-identity/SKILL-CONTEXT.md` if it exists. Two paths for applying brand styling, in order of preference:

### Path 1 — Brand stylesheet (preferred)

If the **Brand stylesheet** field is set, it points to a hand-crafted CSS file (typically at `~/.claude/skills/visual-identity/assets/<brand>.css`). Pass it alongside the bundled default:

```bash
npx --yes md-to-pdf "<name>.md" \
  --stylesheet "$HOME/.claude/skills/md-to-pdf/assets/default.css" \
  --stylesheet "<path from visual-identity Brand stylesheet field>"
```

Order matters — later stylesheets override earlier ones, so the brand CSS wins.

### Path 2 — Generated override (fallback)

If no Brand stylesheet is set but palette and typography fields ARE populated, generate a minimal override on the fly and pass it:

```bash
# Write a brand override CSS from visual-identity tokens
cat > "$TMPDIR_LOCAL/brand.css" <<'EOF'
:root {
  --primary: <Primary hex from visual-identity>;
  --accent:  <Accent hex>;
  --text:    <Text hex>;
}
body { font-family: <Body font stack>, system-ui, sans-serif; color: var(--text); }
h1, h2, h3 { font-family: <Heading font stack>, system-ui, sans-serif; color: var(--primary); }
a { color: var(--accent); }
EOF

npx --yes md-to-pdf "<name>.md" \
  --stylesheet "$HOME/.claude/skills/md-to-pdf/assets/default.css" \
  --stylesheet "$TMPDIR_LOCAL/brand.css"
```

If a logo path is set in visual-identity, reference it via a header image at the top of the markdown (`![](file:///abs/path/to/logo.png)`) or via CSS `background-image` in a cover section of the override CSS.

### PDF cover template

If `visual-identity/SKILL-CONTEXT.md` has a **PDF cover template** path set, generate the body PDF first, then prepend the cover using the `pdf` skill's merge logic:

1. Produce `body.pdf` with md-to-pdf.
2. Hand `body.pdf` + cover-template path to the `pdf` skill with "merge these two".

### If visual-identity is empty or missing — ASK before building

Do NOT silently default. Before producing the PDF, ask:

> "I don't have your visual identity set up yet. Two options — pick one:
> 1. **Bootstrap it now** — I'll grab brand cues from your website (invokes the `visual-identity` skill).
> 2. **Neutral defaults** — I'll use the bundled stylesheet; you can re-style later."

Apply the user's choice before proceeding.

Company identity (name, tagline, URL) for footers/headers comes from ambient memory / CLAUDE.md, or from `~/.claude/skills/identity/SKILL-CONTEXT.md` if set up.

## Prerequisites check

Before the first run, verify Node is available:

```bash
command -v npx >/dev/null || { echo "npx not found — install Node.js first (brew install node or fnm install --lts)"; exit 1; }
```

If missing, tell the user and stop. Don't try to install Node silently.

`md-to-pdf` itself needs no pre-install — `npx` fetches it on first use (takes ~30s the first time, cached after). It pulls Puppeteer which downloads a headless Chromium (~170MB) on first invocation. Warn the user about this on the first run so they know why it's slow.

## Workflow

**Important**: `md-to-pdf` always writes the PDF next to the source `.md` file — there is no `--dest-dir` / `-o` flag. Control the destination by either (a) converting in a working directory and moving the result, or (b) converting in place. Don't waste time looking for an output flag; it doesn't exist.

### Case 1: User provides a file path

```bash
npx --yes md-to-pdf "<path-to-file.md>" --stylesheet "$HOME/.claude/skills/md-to-pdf/assets/default.css"
# PDF appears at <path-to-file>.pdf (same directory, same basename)
# Move it if needed:
mv "<path-to-file>.pdf" "<target-dir>/"
```

### Case 2: User provides inline markdown

Write the content to a temp `.md` file first — `md-to-pdf`'s stdin mode is finicky with complex content, and a temp file gives you a predictable output name. `cd` into the temp dir so the PDF lands there, then move it to the target.

```bash
TMPDIR_LOCAL=$(mktemp -d)
cat > "$TMPDIR_LOCAL/<name>.md" <<'EOF'
<the markdown content>
EOF

(cd "$TMPDIR_LOCAL" && npx --yes md-to-pdf "<name>.md" \
  --stylesheet "$HOME/.claude/skills/md-to-pdf/assets/default.css")

mv "$TMPDIR_LOCAL/<name>.pdf" "<target-dir>/"
```

The `<<'EOF'` heredoc (with quoted `EOF`) prevents shell expansion — important when the markdown contains `$`, backticks, or `!`.

### Case 3: Custom stylesheet

If the user supplies their own CSS path, swap it into `--stylesheet`. If they describe a style ("make it sans-serif", "tighter margins"), edit a copy of the default CSS in the temp dir rather than mutating the bundled one.

## Filename selection

Default order of preference:

1. Explicit `--name` or `--output` style arg from the user
2. Derived from source file (e.g. `notes.md` → `notes.pdf`)
3. First H1 in the content, slugified (e.g. `# Q2 Plan` → `q2-plan.pdf`)
4. Fallback: `document-<YYYYMMDD-HHMM>.pdf`

Ask the user only if nothing above applies.

## Output location

Default to the current working directory (`$(pwd)`). Use `--dest-dir` to force it. If the user wants Desktop or Downloads, pass that path explicitly — don't assume.

## Useful md-to-pdf flags

- `--stylesheet <path>` — CSS file (can pass multiple)
- `--pdf-options '{"format":"A4","margin":{"top":"20mm"}}'` — Puppeteer PDF options as JSON
- `--launch-options '{"args":["--no-sandbox"]}'` — needed in some Linux/container environments
- `--body-class <name>` — adds class to `<body>` for targeted CSS
- `--highlight-style github` — syntax highlighting theme for code blocks
- `--marked-options '{"gfm":true}'` — GitHub-flavored markdown (already default)

Full reference: https://github.com/simonhaenisch/md-to-pdf

## After conversion

1. Confirm the PDF exists (`ls -lh <output>.pdf`)
2. Report the absolute path to the user so they can open it
3. On macOS, offer to open it: `open <output>.pdf`

## Failure modes to watch for

- **Chromium download fails** (corporate network, firewall) — tell the user to set `PUPPETEER_SKIP_DOWNLOAD=false` or use an existing Chrome via `--launch-options '{"executablePath":"/path/to/chrome"}'`
- **Empty PDF** — usually means the markdown was empty or a heredoc got mangled. Check the temp `.md` file before converting.
- **Huge PDF from a small file** — embedded images are likely unoptimized. Note it but don't auto-fix unless asked.
- **`npx` prompts to install** — always pass `--yes` to suppress the prompt.

## Example session

User: "can you turn these meeting notes into a pdf?" [pastes markdown]

1. Write notes to `/tmp/tmp.XXXX/meeting-notes.md`
2. Run `npx --yes md-to-pdf` with the bundled stylesheet, `--dest-dir "$(pwd)"`
3. Report: "Created `meeting-notes.pdf` at /Users/.../meeting-notes.pdf (214KB). Want me to open it?"
