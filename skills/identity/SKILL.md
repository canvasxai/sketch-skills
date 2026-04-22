---
name: identity
description: >
  Read or bootstrap the org's identity — company name, one-liner, audience,
  primary URL, social handles, pricing stance, flagship products. Can
  auto-discover all of this from the user's email domain on first run.
  Trigger when the user asks: "what do we know about our company", "what's
  our company profile", "who are we", "set up our company profile",
  "bootstrap our org info", "find our company info", "discover our identity
  from our website", "refresh our company info", "update our company
  description". Also invoke this skill as a prerequisite when another skill
  needs identity data and `~/.claude/skills/identity/SKILL-CONTEXT.md` is
  empty or missing. Do NOT trigger for brand voice, tone, palette, fonts,
  or logos — those are handled by `voice-and-tone` and `visual-identity`.
---

# Identity

This skill holds (and can auto-discover) the facts about the user's own company — the data that answers "who are we". It does not hold voice, palette, or logo; those live in `voice-and-tone` and `visual-identity`.

Data is persisted two ways:

1. **`SKILL-CONTEXT.md`** in this directory — authoritative, user-readable, user-editable.
2. **Auto-memory** (project-type entry) — ambient, loaded into every future session.

When both are present, the file wins (the user may have corrected it since).

## When invoked

### If `SKILL-CONTEXT.md` is already populated

Read it and answer the user's question directly. If they want to update a specific field, edit the file and also update the corresponding auto-memory entry so the ambient copy stays in sync.

### If `SKILL-CONTEXT.md` is empty or missing — auto-discover

1. Extract the domain from the user's email (e.g. `alex@acme.com` → `acme.com`).
2. Tell the user what you're about to do, using their actual domain:
   > "Let me see what I can find from `<their-domain>` — one sec."
3. Use the `canvas` skill to:
   - Scrape the homepage: `$CANVAS_CLI direct-execute-web-scrape --url "https://<domain>" --formats markdown --only-main-content true --output json`
   - Scrape `/about` if it exists.
   - Scrape `/pricing` if it exists (signals pricing stance + tier names).
   - Web-search for social handles: `$CANVAS_CLI direct-execute-web-search --query "<domain> twitter linkedin" --limit 3 --output json`
4. Extract from scraped content:
   - **Company name** (brand name as they write it, not the domain)
   - **One-liner** (homepage hero subheadline, or `<title>` if cleaner)
   - **Audience** (who the homepage says the product is for)
   - **Primary URL**
   - **Social handles** (Twitter/X, LinkedIn, others if prominent)
   - **Pricing stance** (free / freemium / paid / enterprise-only / "contact sales") — and the `/pricing` URL if found
   - **Flagship products/features** (top 3–5, if the homepage lists them)
5. **Summarise back to the user** in a short block substituting their actual company and domain, then ask for confirmation. Example shape (do not use these values — they are illustrative):

   > "Here's what I pulled from `<their-domain>` — please flag anything wrong before I save it:
   >
   > - **Company:** <name as written on the homepage>
   > - **One-liner:** <from the hero or meta description>
   > - **Audience:** <inferred from the homepage>
   > - **URL:** https://<their-domain>
   > - **Social:** <handles found>
   > - **Pricing:** <stance from /pricing>
   > - **Flagship products:** <top 3–5 from the homepage>
   >
   > Save this as our company profile?"
6. On confirmation: write to `SKILL-CONTEXT.md` *and* save a project-type auto-memory entry (`identity_org.md`) with the same facts. If the user corrects fields first, persist the corrected version.
7. If a page can't be scraped (blocked, 404, login-walled), skip that source and note it — don't get stuck.

### If a source is ambiguous

If there are multiple plausible matches (multiple companies on the same domain root, or multiple candidate one-liners), present the options and ask. Never guess.

## Updating fields later

When the user asks "update our company description to X" or "our new URL is Y": edit the matching field in `SKILL-CONTEXT.md`, update the auto-memory entry, and summarise the diff in one line. No need to re-run the full discovery flow.

## Refreshing from source

If the user says "refresh" or "check if anything's changed", re-run the discovery flow and diff the results against the current file. Present the diff, ask before overwriting.

## Authoring rules (MUST follow when writing to SKILL-CONTEXT.md)

1. **Strip placeholder text completely.** The template uses HTML comments (`<!-- ... -->`) for hints. Never leave hint text inside a value.
2. **Only record what the source actually confirmed.** Don't invent a one-liner that the homepage didn't say. If a field is unknown after discovery, leave it blank and flag it to the user rather than guessing.
3. **Stamp `Last verified` on every change.** Date in `YYYY-MM-DD` format. Future sessions use this to decide whether to trust or refresh.
4. **Don't rename sections or field labels.** Consumer skills (icp-discovery in particular) pattern-match on them.
5. **Flagship products: top 3–5 only, in homepage order.** Don't dump the entire product catalog.
