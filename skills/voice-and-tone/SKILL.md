---
name: voice-and-tone
description: >
  Read or update the org's voice-and-tone guide — default tone, sound-like
  references, banned words and phrases, house examples, product vocabulary,
  capitalisation rules. Trigger whenever the user asks about or wants to
  change how the company sounds in writing. Read triggers: "what's our tone",
  "what's our brand voice", "what words do we avoid", "what's our voice
  guide", "how do we write", "show me our voice rules". Update triggers:
  "add a banned word", "update our tone", "save this as our voice",
  "remember that we prefer X", "add this to our voice guide", "we never say
  Y". Do NOT trigger this skill for producing copy, decks, docs, PDFs,
  research briefings, or ICP discovery — those skills read the voice file
  on their own when they run.
---

# Voice and Tone

This is a **data skill**: the org's voice-and-tone guide lives in `SKILL-CONTEXT.md` in this directory. Consumer skills (copywriter, landing-page-copy, pptx, research, icp-discovery) read it at session start to keep output on-voice.

Once invoked: read `SKILL-CONTEXT.md` and either answer the user's question from it or edit it to reflect the update. On sweeping changes (full voice overhaul), show a preview before writing. On single-field edits (add one banned word, tweak one rule), just apply it and summarise in one line.

## Authoring rules (MUST follow when writing to SKILL-CONTEXT.md)

1. **Strip placeholder text completely.** The template uses HTML comments (`<!-- ... -->`) for hints. Never leave hint text inside a value.
2. **Banned words list: one entry per line, just the word or phrase.** No explanations inline. If a ban needs rationale, add it as a sub-bullet beneath the word.
3. **House examples are verbatim.** Paste the actual line, no paraphrasing. Don't add running commentary inside a block quote.
4. **Don't rename sections or field labels.** Consumer skills pattern-match on them. Extend by adding new fields; don't restructure existing ones.
5. **Vocabulary rows: use the existing columns (`Term | Write as | Never write as`).** If the brand has more rules than the columns support, add more rows — not more columns.

If the user wants to add a whole new section that isn't in the schema, add it. The schema is a starting point, not a cap.
