---
name: research
description: >
  Research a person, company, product, topic, trend, or event and produce a
  grounded briefing. Use this skill whenever the user wants to look into or
  learn about something they don't already know deeply. Triggers include
  "research X", "look into X", "what's the latest on X", "find out about X",
  "who is X", "what does X do", "dig into X", "give me background on X",
  "what are people saying about X", "what's the sentiment on X", "how is X
  doing", "quick brief on X", "deep dive on X", "tell me about [company or
  person]", "prep me for a meeting with X", "who am I meeting next",
  "who is my next meeting with", "check my calendar and research". Use even
  when the user doesn't say the word "research" — if they're asking to learn
  about an entity or topic they're unfamiliar with, that's research.
category: research
---

# Research

Produce a grounded, actionable briefing. The user should have the right context in 1–3 minutes of reading. Don't narrate searches — just go.

## Brand context

At the start of every session, read `~/.claude/skills/voice-and-tone/SKILL-CONTEXT.md` if it exists. Apply tone defaults, vocabulary rules, and banned words to the briefing's prose (bullets and synthesis) — but not to direct quotes from sources, which are preserved verbatim.

Research briefings are factual, so a missing voice file is less critical here — **don't block the brief to ask**. Use neutral factual prose if the file is empty. If the user's prior briefs have had consistent voice feedback, mention once at the end: "Want me to save a voice rule for future research briefs?"

Company identity (name, products) — for prep briefings that mention us — comes from ambient memory / CLAUDE.md, or from `~/.claude/skills/identity/SKILL-CONTEXT.md` if set up.

## Canvas CLI

All web and app lookups go through `$CANVAS_CLI`. Always pass `--output json`. Do not mention Canvas to the user.

```bash
# Web search
$CANVAS_CLI direct-execute-web-search --query "..." --limit 5 --output json

# Scrape a URL (public pages only — not login-walled apps)
$CANVAS_CLI direct-execute-web-scrape --url "https://..." --formats markdown --only-main-content true --output json

# LinkedIn recent activity (if connected)
$CANVAS_CLI get-components --apps "linkedin" --component-type action --q "get recent activity" --limit 5 --output json
# Then: $CANVAS_CLI direct-execute-action --component-key <key> --configured-props '{"profileUrl":"..."}' --output json

# Reddit (if connected)
$CANVAS_CLI get-components --apps "reddit" --component-type action --q "search posts" --limit 5 --output json
# Then: $CANVAS_CLI direct-execute-action --component-key <key> --configured-props '{"query":"...","subreddit":"..."}' --output json
```

If `$CANVAS_CLI` is unset, call `getProviderConfig` and tell the user integrations aren't configured.

**Pipe rule:** Never pipe `$CANVAS_CLI` directly. Use `sh -c '"$CANVAS_CLI" ...' | jq ...` if post-processing is needed.

**Errors:**
- `CONNECTION_NOT_CONNECTED` → tell the user the app isn't connected, fall back to web search
- `(eval):1: permission denied:` → retry with the `sh -c` wrapper

## Calendar-first workflow (for "prep me for my next meeting" or "who am I meeting")

When the user asks about an upcoming meeting or wants to prep for a call, **check the calendar first**, then research the person(s).

```bash
# 1. Confirm Google Calendar is connected
$CANVAS_CLI search-apps --queries "google_calendar" --output json

# 2. Fetch the current user's timezone and account
$CANVAS_CLI direct-execute-action --component-key google_calendar-get-current-user --configured-props '{}' --output json

# 3. List upcoming events (use current UTC time for timeMin, singleEvents:true for recurring)
$CANVAS_CLI direct-execute-action \
  --component-key google_calendar-list-events \
  --configured-props '{"calendarId":"primary","maxResults":5,"timeMin":"<ISO8601_UTC>","orderBy":"startTime","singleEvents":true}' \
  --output json
```

**Identify who to research:**
- For 1:1 meetings: research the other attendee.
- For group meetings (standup, all-hands, recurring syncs with internal team only): skip research and just surface the event details.
- For meetings with external attendees mixed with internal: focus research on the external person(s).
- Internal vs. external: check email domains. Internal = same domain as the user (the user's email domain is available in the system context). External = everyone else.

**Extract research targets from the event:**
- Prefer the `description` field — booking tools (Calendly, Cal.com, Avail) often embed the invitee's name, email, company, and context notes there.
- Fall back to `attendees[].email` for external addresses.
- Use the event `summary` (title) for context about the meeting purpose.

**Then run the standard research flow** on the identified person(s), using name + company + email as search seeds.

**Calendar errors:**
- If `google_calendar-list-events` fails with auth error but `google_calendar-get-current-user` succeeds, use the correct component key `google_calendar-list-events` (with underscore slug, not hyphenated app prefix). Both are valid — try the underscore slug form if the hyphen form fails.
- If Google Calendar isn't connected: tell the user to connect it in Settings → Integrations.

## Depth modes

Read the ask and pick a mode:

- **Quick** — 1–2 sources, 3–5 bullets in chat. Default for conversational asks.
- **Standard** — 2–3 sources, structured briefing in chat.
- **Deep** — 3–5 sources, save to `output/research-{slug}.md`, deliver via `SendFileToChat`.

For calendar-triggered prep briefs, default to **Standard** unless the user signals they want more depth.

If unclear, ask one question: *"Quick read or deeper prep for something specific?"* Skip it if the ask already signals depth or purpose.

## Source selection

Match sources to the entity — don't use a source just because it exists.

| Entity | Primary | Add if needed |
|---|---|---|
| Person | Web search + LinkedIn activity | Reddit (public figures), Gmail/Notion (prior history) |
| Company | Scrape homepage + /about + /pricing + web news search | Reddit (sentiment), LinkedIn (team signals) |
| Product | Product site + web search for reviews | Reddit threads for real user sentiment |
| Topic/trend | Web search + Reddit | LinkedIn posts, scrape 1–2 authoritative pages |
| Event | Web search with date qualifier + scrape primary source | LinkedIn reactions |

**Two-pass for Standard/Deep:** broad search first → scrape the 2–3 most relevant URLs for depth.

## Briefing format

```
*{Entity}* — {one-line characterization}

*TL;DR*
{2–3 sentences}

*Key facts*
- {Specific fact with number/date/name}

*What people are saying*
- {Quote or synthesis, attributed: "per r/sub", "per LinkedIn", "per TechCrunch"}

*Signals worth noting* (optional)
- {Funding, hiring, controversy, trigger event}

*Sources*
- {Publication/platform names, no raw URLs}
```

Adapt to the entity. For a person, swap "Signals" for "Recent posts". For a calendar-triggered brief, add a *Meeting context* section above TL;DR:

```
*Meeting context*
- Event: {title}, {date} at {time} ({timezone})
- Duration: {X} minutes
- Format: {Google Meet / in-person / phone}
- Their stated reason for the call: {from description field, if available}
```

Don't apply the template mechanically.

## Writing rules

- Specifics beat abstractions. "Raised $32M Series B in March 2026 led by Accel" > "recently raised funding".
- Quote real people when possible. Attribute everything.
- Flag weak signals explicitly ("per one Reddit thread — unverified").
- Never invent dates, numbers, names, or quotes.
- If a page is paywalled or blocked: one try, note it, move on.
- Ambiguous entity (common name, overlapping products): ask before searching.
- Conflicting sources: surface the contradiction, don't silently pick a winner.

## Repeat asks → offer a watch

If the user asks about the same entity a second time:
> "Want me to keep an eye on {entity} and ping you weekly if anything changes?"

On yes, use `ManageScheduledTasks` to set up a weekly run, delivered as a DM.
