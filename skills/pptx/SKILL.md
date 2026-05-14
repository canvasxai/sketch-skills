---
name: pptx
description: >
  Create, read, and edit PowerPoint slide decks (`.pptx` files). Use this skill
  any time the user mentions "deck", "slides", "presentation", "PowerPoint",
  "pitch deck", "PPT", or attaches a `.pptx` file. Triggers: "build me a deck
  about X", "make slides for…", "turn this into a pitch deck", "update slide 3",
  "add a chart to my deck", "read this presentation", "summarise these slides",
  "make a 10-slide intro deck for our team all-hands". Use even if the user only
  says "deck" or "slides" without naming a file format — they mean PowerPoint.
  Do not mention `python-pptx` or other internal libraries to the user.
category: productivity
---

# PPTX

## What this skill does

Builds, reads, and edits PowerPoint decks. Two modes:

1. **Read/edit an existing deck** — the user attached a `.pptx`, wants content extracted, or wants specific slides changed.
2. **Build a new deck** — the user describes what they want and you produce a finished `.pptx`.

Use `python-pptx` for programmatic slide creation and editing. For complex layouts or brand-heavy designs, prefer the HTML-first approach (write HTML/CSS, convert to PPTX).

## How files flow in Sketch

- **Incoming**: attached decks land in `attachments/<filename>.pptx`. Read from there.
- **Outgoing**: write produced decks to `output/<slug>.pptx` and deliver via `SendFileToChat`.
- **Images/charts**: generate them in the workspace (matplotlib / PIL / Chart.js render), reference from slides.

## Brand context

Decks have both copy and visuals, so at the start of every session read both of these if they exist:

- `~/.claude/skills/voice-and-tone/SKILL-CONTEXT.md` — apply tone defaults, banned words, house examples, and vocabulary rules to slide copy (titles, body text, speaker-note cues).
- `~/.claude/skills/visual-identity/SKILL-CONTEXT.md` — apply the palette to slide backgrounds, accents, shapes, and chart colors; use the typography stack for heading and body fonts; place the logo on the title slide and footer; pair the tagline with the logo if set. If an assets-root path is set, pull icons and imagery from there before generating new ones.

### Templates (preferred path for branded decks)

If `visual-identity/SKILL-CONTEXT.md` has a **Deck template (.pptx)** path set, open that template and insert content into its masters and layouts. This gives near-native brand quality:

```python
from pptx import Presentation
prs = Presentation("<deck template path from visual-identity>")
slide = prs.slides.add_slide(prs.slide_layouts[1])  # use layouts from the template
# Fill placeholders by index — styling inherits from the master
slide.placeholders[0].text = "Title"
slide.placeholders[1].text = "Body"
```

Rules when using a template:
- Use `prs.slide_layouts[N]` — never `Presentation()` with a blank. The template's masters define colors, fonts, and placeholder styling.
- Fill placeholders by index or name (`slide.placeholders[idx]`). Don't add free-standing text boxes unless the layout genuinely lacks a placeholder.
- For charts/shapes, use theme colors (`MSO_THEME_COLOR.ACCENT_1` etc.) so the template's palette applies.
- Don't try to restyle master slides programmatically — if a layout is wrong, report it to the user, don't hack around it.

### If visual-identity is empty or missing — ASK before building

Do NOT silently default. Before producing the deck, ask the user:

> "I don't have your visual identity set up yet. Three options — pick one:
> 1. **Bootstrap it now** — I'll grab brand cues from your website (invokes the `visual-identity` skill).
> 2. **Use a template** — if you have a branded `.pptx` template, upload it to the workspace or point me at a path and I'll build into it.
> 3. **Neutral defaults** — I'll build with a neutral palette and system fonts; you can re-style later."

Apply the user's choice before proceeding. Always offer option 2 at the end even if the user chose option 3 — "Done. Want me to re-style this with a brand template if you upload one?"

### Existing decks

When **reading or editing** an existing deck, don't restyle it — preserve the author's theme. Only apply brand styling to net-new slides or full rebuilds.

Company identity (name, tagline, URL) comes from ambient memory / CLAUDE.md, or from `~/.claude/skills/identity/SKILL-CONTEXT.md` if set up.

## Building a new deck — the conversation flow

Unless the user has given you a complete brief (audience + purpose + length + key messages), ask **one** clarifying message that covers the minimum:

> "Before I build this — three quick things:
> 1. Who's the audience? (internal team, investors, customers, etc.)
> 2. How long? (rough slide count)
> 3. What's the single message they should walk away with?"

Then propose an outline (slide-by-slide titles + one-line each) in chat for confirmation **before** building the file. This saves rebuilding after the user sees the wrong structure.

Default length if the user doesn't specify: 8–12 slides. Default structure:
- Title
- The one thing (the single message restated)
- Context / problem
- Insight / approach
- Evidence (2–3 slides of proof, data, examples)
- Implication / what it means
- Ask / next step

## Building with python-pptx

```python
from pptx import Presentation
from pptx.util import Inches, Pt
from pptx.dml.color import RGBColor

prs = Presentation()
prs.slide_width = Inches(13.333)
prs.slide_height = Inches(7.5)  # 16:9

# Title slide
slide = prs.slides.add_slide(prs.slide_layouts[0])
slide.shapes.title.text = "One AI assistant for your team"
slide.placeholders[1].text = "Deploy once. Show up everywhere."

# Content slide
slide = prs.slides.add_slide(prs.slide_layouts[1])
slide.shapes.title.text = "The problem"
body = slide.placeholders[1].text_frame
body.text = "Every teammate sets up their own AI."
for point in ["Their own API keys", "Their own context", "Nothing shared"]:
    p = body.add_paragraph()
    p.text = point
    p.level = 1

prs.save("output/deck.pptx")
```

**For brand-styled decks** — load a template (`.pptx` with master slides already styled) and insert content into its layouts, instead of building from `Presentation()` blank:

```python
prs = Presentation("assets/template.pptx")  # bundled brand template
slide = prs.slides.add_slide(prs.slide_layouts[1])
# placeholders already styled; just fill text
```

If the user's org has uploaded a brand template to the workspace, use it. Otherwise build neutral and ask at the end whether they want it re-styled.

## Reading / summarising a deck

```python
from pptx import Presentation

prs = Presentation("attachments/deck.pptx")
for i, slide in enumerate(prs.slides, 1):
    title = slide.shapes.title.text if slide.shapes.title else "(no title)"
    bullets = []
    for shape in slide.shapes:
        if shape.has_text_frame:
            for p in shape.text_frame.paragraphs:
                if p.text and p.text != title:
                    bullets.append(p.text)
    print(f"Slide {i}: {title}\n  " + "\n  ".join(bullets))
```

When summarising a deck in chat, group by section: "Here's what's in the deck — sections are Intro (slides 1–2), Problem (3–5), Solution (6–9), Proof (10–12), Ask (13)."

## Editing an existing deck

When the user says "change slide 3 to…" or "add a slide after the pricing one", treat it as: **load → locate target → mutate → save to a new filename**. Don't overwrite the attached original. Name the new file descriptively (e.g., `deck-updated.pptx` or `deck-v2.pptx`).

```python
prs = Presentation("attachments/original.pptx")
slide_3 = prs.slides[2]
slide_3.shapes.title.text = "New title"
prs.save("output/deck-updated.pptx")
```

## Adding charts

Use `python-pptx` chart API for native editable charts (not images):

```python
from pptx.chart.data import CategoryChartData
from pptx.enum.chart import XL_CHART_TYPE

chart_data = CategoryChartData()
chart_data.categories = ["Q1", "Q2", "Q3", "Q4"]
chart_data.add_series("Revenue", (100, 140, 180, 230))

slide.shapes.add_chart(
    XL_CHART_TYPE.COLUMN_CLUSTERED,
    Inches(2), Inches(2), Inches(8), Inches(4.5),
    chart_data,
)
```

Prefer native charts over screenshot images — the user can edit them in PowerPoint later.

## Images

If the user asks for visuals and none are provided, generate placeholder images in the workspace (PIL, matplotlib) or leave styled empty blocks with a note in chat: "I left placeholders for the product screenshots on slides 4 and 7 — drop them in and they'll size correctly." Never generate fake data, fake logos, or fake people.

## Reply format

- After building: one-sentence description + `SendFileToChat` with the `.pptx`. Example: "Built a 10-slide deck — title, problem, approach, three proof slides, ask. File below."
- After reading: structured summary in chat (sections with slide numbers), never full slide-by-slide dumps unless asked.
- After editing: brief diff-style summary ("Updated slide 3 title, added a new slide 6 with the pricing table") + updated file.

## Error handling

- **Corrupt or non-standard `.pptx`** → try loading anyway; if it fails, tell the user briefly and ask if they can re-export from PowerPoint.
- **Missing fonts** → the deck still builds, just with font substitution. Mention this if it matters.
- **Never mention** "python-pptx" or library internals to the user.
