---
name: visual-identity
description: >
  Read or update the org's visual identity — palette (hex colors), typography
  (fonts), logo paths, tagline, assets root directory. Trigger whenever the
  user asks about or wants to change how the company looks. Read triggers:
  "what's our primary color", "what fonts do we use", "where's our logo",
  "what's our tagline", "show me our palette", "what's our visual identity",
  "do we have brand colors set up". Update triggers: "change the primary
  color to #FF00AA", "update our heading font to Inter", "update our logo
  path", "we rebranded — here's the new palette", "set our assets folder
  to /path". Do NOT trigger this skill for producing decks, docs, or PDFs
  themselves — those skills (pptx, docx, pdf) read the visual file on their
  own when they run. Do NOT trigger for voice/tone/copy matters — those are
  handled by `voice-and-tone`.
---

# Visual Identity

This is a **data skill**: the org's visual identity lives in `SKILL-CONTEXT.md` in this directory. Consumer skills (docx, pdf, pptx, landing-page-copy, md-to-pdf) read it at session start when they're producing branded output.

Once invoked: read `SKILL-CONTEXT.md` and either answer the user's question from it or edit it to reflect the update. On sweeping changes (full rebrand), show a preview before writing. On single-field edits (swap one hex, change one font), just apply it and summarise in one line.

## Authoring rules (MUST follow when writing to SKILL-CONTEXT.md)

These prevent the common failure modes where downstream skills can't parse the data:

1. **Strip placeholder text completely.** The template uses HTML comments (`<!-- ... -->`) for field-level hints. Never leave hint text inside a value. Never preserve italic wrappers like `_(…)_` from older templates — replace the whole value.
2. **Values are values; annotations go in Notes.** Don't stuff guidance into a value line. Example wrong: `Heading font: Inter, sans-serif (use bold + uppercase for H1)`. Example right: put the font stack in the value, put the "bold + uppercase" rule in a separate `Heading casing:` field or a Notes column.
3. **File paths must be absolute LOCAL paths, not URLs.** pptx and docx can't embed images from HTTPS — they need a local file. If the source is a URL (e.g. a Supabase-hosted logo), download it into `~/.claude/skills/visual-identity/assets/<filename>` via the `canvas` skill's `direct-execute-web-scrape` or a plain `curl`, then record the local absolute path. Keep the original URL in an HTML comment above the field for provenance.
4. **Preserve the canonical palette rows.** The template ships with `Primary`, `Background`, `Text`, `Accent` rows so consumer skills can programmatically look up `palette[primary]`. Don't rename these rows. If the brand thinks of colors differently (`Background / Surface / Border / Body Text`), add the brand-specific rows **below** the canonical four, with their own names — don't replace.
5. **Extend sections by adding fields, not by renaming.** If you need a field the template doesn't have, append it as a new bullet. Never rename existing sections or field labels — downstream skills pattern-match on them.
6. **Always update the Last verified line if present.** Stamps help the agent know how stale the data is.

If the user wants to add a whole new section that isn't in the schema, add it — the schema is a starting point, not a cap. But don't rename or reorder existing sections.
