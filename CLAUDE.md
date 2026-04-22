# sketch-skills — maintenance guide

This repo holds the skills that Sketch servers auto-sync into each workspace's `~/.claude/skills/` on startup (see `packages/server/src/skills/sync.ts` in the `sketch` repo). Treat every file here as **public-generic** — no user or org data, ever.

## The model: one source of truth + per-org overlay

Each skill has two files that matter:

| File | Who owns it | Synced by Sketch on startup? |
|---|---|---|
| `<skill>/SKILL.md` | This repo. Generic. Same for every install. | **Yes** — copied into `~/.claude/skills/<skill>/SKILL.md` every restart. Overwrites local. |
| `<skill>/SKILL-CONTEXT.md` (data skills only) | The user. Per-org data. | **Seeded once** on first install. Not overwritten on subsequent syncs — user edits persist. |

So: **edit `SKILL.md` here. Edit `SKILL-CONTEXT.md` locally** (or via the skill's own update triggers). Changes to `SKILL.md` land on every install on their next restart. Changes to `SKILL-CONTEXT.md` live only in that install.

No scrubbing step. No two-copy sync. No grep-for-secrets. Org-specific data never goes into `SKILL.md` — it goes into the user's `SKILL-CONTEXT.md`.

## Skill categories

Three kinds of skills ship from this repo (and the broader local set):

### 1. Tool / action skills
Wrap a specific set of tools or an external system. Fire when the user wants to *do* something with that system.

- `canvas` — gateway to 3000+ apps via `$CANVAS_CLI`.
- `clickup` — ClickUp task/project management. Has its own `SKILL-CONTEXT.md` for stable IDs (workspaces, users, lists, spaces, folders) the agent accumulates.

### 2. Data skills
Hold org-specific context that other skills read. A data skill's body is thin — its value is in `SKILL-CONTEXT.md`. Three ship here:

- `identity` — company name, one-liner, audience, URL, socials, pricing stance, flagship products. Auto-discovers from the user's email domain on first run.
- `voice-and-tone` — tone defaults, sound-like references, banned words, house examples, vocabulary.
- `visual-identity` — palette, typography, logo paths, tagline, brand stylesheet, templates.

### 3. Content-producing skills
Generate user-facing artifacts. They read the data skills' `SKILL-CONTEXT.md` at session start and apply what's populated.

- `icp-discovery` — reads `identity` + `voice-and-tone`.
- `md-to-pdf` — reads `visual-identity`. Canonical path for creating PDFs from content.
- Local-only (not in this repo but wired to the same data skills): `copywriter`, `landing-page-copy`, `research`, `docx`, `pdf`, `pptx`.

## Writing a skill description (the frontmatter)

The `description` field in a skill's frontmatter is **what the router reads to decide whether to trigger**. The body is only for the agent once the skill is already loaded.

Rules:

- **List explicit trigger phrases** the user might say, in quotes. "make a pdf", "what's our tone", etc. Don't rely on the agent inferring; spell it out.
- **List explicit anti-triggers** when there's a neighboring skill that overlaps. Example: `md-to-pdf` says "Do NOT use for manipulating existing PDFs — use the `pdf` skill". `pdf` says the mirror.
- Keep it **agent-facing**, not user-facing. "Use this skill when…" is fine. Marketing copy is not.
- Distinct read vs update triggers for data skills (see `voice-and-tone`, `visual-identity`, `identity` descriptions).

## Data skill conventions

When building a new data skill (anything that holds org-specific config):

1. **Ship two files**: `SKILL.md` (generic, body is thin) and `SKILL-CONTEXT.md` (template with placeholders).
2. **Use HTML comments for hints, not italic placeholders.** Agents that read the template and try to fill it in will preserve whatever ambiguous markup is there — `_(hint text)_` frequently ends up left inside the value. HTML comments (`<!-- ... -->`) are visible to the agent but can never be mistaken for values.
3. **Include an `## Authoring rules` section in SKILL.md.** Explicit instructions for any agent that will write to the CONTEXT file: strip placeholders, separate values from annotations, use absolute local paths, preserve canonical field labels, stamp `Last verified`. See `visual-identity/SKILL.md` for the current example.
4. **Include a mirror of the authoring rules at the top of the CONTEXT template** as an HTML comment, so agents that skim only the CONTEXT file still see the rules.
5. **Canonical rows.** If consumer skills will programmatically look up a field (e.g. `palette[primary]`), reserve that row in the template. Instruct agents to add brand-specific rows below the canonical ones rather than renaming.

## Consumer skill conventions

Any skill that produces user-facing output (copy, docs, decks, PDFs, briefings) should:

1. **Read the relevant data skills' `SKILL-CONTEXT.md` at session start.** Map them in a "Brand context" section near the top of the SKILL.md body.
2. **Use the prompt-on-empty pattern** when a data file is empty and the output needs it. Don't silently default. Offer numbered options: bootstrap, use a template, neutral defaults. See `pptx/SKILL.md` or `md-to-pdf/SKILL.md` for the current phrasing.
3. **Don't duplicate data across skills.** If `identity` already has the company name, the consumer skill reads it — it doesn't copy it into its own CONTEXT file. Memory + data skills are the sources of truth.

## Cross-skill composition

Skills frequently compose. Patterns in use:

- **Bootstrap-on-empty**: `icp-discovery` invokes `identity` if `identity/SKILL-CONTEXT.md` is empty.
- **Create-then-manipulate**: `md-to-pdf` produces the body PDF; if `visual-identity` has a PDF cover template, the body is handed to `pdf` for a merge.
- **Template-over-scratch**: `pptx` and `docx` prefer loading a branded template (from `visual-identity/SKILL-CONTEXT.md`'s Templates section) over building from a blank canvas.

When designing a new skill, ask: is there an existing data skill whose file I should read instead of asking the user? Is there a neighboring action-skill that should own the other half of the flow?

## Maintenance workflow

Much simpler than the old scrub-and-mirror model:

1. **Edit `skills/<skill>/SKILL.md` in this repo** directly. No local-first requirement — there's no user data in SKILL.md to protect.
2. **For data-skill templates**: edit `skills/<data-skill>/SKILL-CONTEXT.md` here when the *schema* changes. Never put real data in this repo's SKILL-CONTEXT.md — the per-install file is untouched by the sync after first boot, but the repo's copy is what new installs get seeded with.
3. **Mirror to your own local install** for testing: `cp skills/<skill>/SKILL.md ~/.claude/skills/<skill>/SKILL.md`. Don't overwrite your local `SKILL-CONTEXT.md` — it has your org's data.
4. **PR on a feature branch**; never commit directly to `main`. PR description should flag:
   - Schema changes to `SKILL-CONTEXT.md` (existing installs still have the old schema in their local copy — they won't auto-upgrade).
   - Description-field changes (affects routing across all installs on next restart).
   - CLI-drift additions (see below).
5. **After merge**, `git pull` the local clone so your `~/.sketch/skills-repo/` cache is current.

**Sync caveat to still be aware of:** Sketch overwrites `SKILL.md` on each restart but preserves `SKILL-CONTEXT.md` if it already exists. Schema additions to CONTEXT files therefore don't auto-land on existing installs — the user keeps their old schema until they delete the local file or merge the new sections in manually.

## CLI-drift awareness

`canvas` and `clickup` SKILL.md files are tightly coupled to `canvas-cli.js`. When the CLI bundle changes:

- Pull the latest `canvas-cli.js` into `skills/canvas/canvas-cli.js` from the source of truth.
- Sanity-check the subcommands referenced in every SKILL.md against `node canvas-cli.js --help`. Flag unknown-command failures in a follow-up PR.
- Recent rename examples (keep an eye out for more):
  - `get-app-components` + `bulk-get-components` → merged into `get-components --apps <slug-or-csv>`
  - `clickup-get-tasks` + `clickup-get-tasks-by-user` + `clickup-get-tasks-in-space` → merged into `clickup-find-tasks`
  - Added: `clickup-context`, `clickup-project-summary`

## Related context

- Canonical local skill paths: `~/.claude/skills/<skill>/SKILL.md` + `~/.claude/skills/<skill>/SKILL-CONTEXT.md` (data skills only).
- Sketch's sync code: `packages/server/src/skills/sync.ts` in the `canvasxai/sketch` repo. The current sync overwrites `SKILL.md` on every restart but preserves `SKILL-CONTEXT.md` after the first seeding — that's the contract this repo's layout depends on.
- SDK Bash-tool pipe block and the `sh -c` wrapper workaround (used throughout `canvas` and `clickup`): `.planning/CANVAS_WRAPPER_AGENT_SELF_DEBUG.md` and `.planning/WORKFLOW-SESSION-HANDOFF.md` in the `sketch` repo.
