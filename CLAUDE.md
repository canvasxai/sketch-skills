# sketch-skills — maintenance instructions

This repo hosts the featured skills (`canvas`, `clickup`, `icp-discovery`) that Sketch servers auto-sync into `~/.claude/skills/` on startup (see `packages/server/src/skills/sync.ts` in the `sketch` repo). Treat every file here as **public-generic**. No user or org data in this repo, ever.

## Two-copy model

There are two live copies of each SKILL.md:

1. **Local (user-specific):** `~/.claude/skills/<skill>/SKILL.md`
   - Used by the maintainer's local Sketch instance (sketch-test).
   - Contains real IDs, org names, user names, absolute paths — deliberately. This makes the local agent fast: no re-discovery of stable IDs, no tool-call round-trips to find the user or the workspace.
   - This is the authoritative source for *content changes*. Edit here first.

2. **Repo (generic):** `<this repo>/skills/<skill>/SKILL.md`
   - Distributed to every Sketch install via `syncFeaturedSkills`.
   - MUST be sanitized — see "User/org-specific content to strip" below.
   - Structure, guidance, examples with placeholders — yes. Real IDs — no.

**Sketch's sync behavior (as of 2026-04-21):** `syncFeaturedSkills` skips any skill directory that already exists in `~/.claude/skills/`. So once a user's local copy diverges (because they've filled in Known IDs, added workflows, etc.), upstream updates do NOT clobber it. This means: **bug fixes and new CLI-API changes landed here won't reach existing installs** unless the user deletes their local skill dir and restarts. Document breaking changes in PR descriptions so users know when to refresh.

## Maintenance workflow

Standard loop:

1. Edit the skill locally at `~/.claude/skills/<skill>/SKILL.md`. Test it against sketch-test until you're happy.
2. Copy the file into this repo at `skills/<skill>/SKILL.md`.
3. Sanitize — remove everything in the "User/org-specific content to strip" list below. Use the `rg` / `grep` verification command in "Verification" before committing.
4. Open a PR on a feature branch (never commit direct to `main`). PR description should call out:
   - What CLI version the doc assumes (so future maintainers can diff against the shipping `canvas-cli.js`).
   - Whether the change is additive (safe) or removes/renames actions (breaking — mention that existing installs won't auto-pick-up).
5. After merge, `git pull` in the local clone to stay synced.

## User/org-specific content to strip

Any time you mirror local → repo, scrub:

- **ClickUp workspace team_id**: `9002212861` → `<team_id>`
- **ClickUp workspace name**: `Habuild - Yoga Everyday` → remove or replace with placeholder
- **Maintainer user_id**: `100817722` → placeholder row or remove
- **Maintainer name**: `Himanshu Kalra` → remove or replace
- **Specific list_ids**: `901613959985` (Sketch Roadmap) → placeholder row or remove
- **Absolute home-directory paths**: `/Users/hkalra/.claude/...` → `~/.claude/...`
- **ClickUp URL examples embedding real IDs**: `app.clickup.com/9002212861/...` → `app.clickup.com/<team_id>/...`
- **Known IDs tables**: keep the section structure (headings, column names, guidance paragraph) but replace data rows with `_(add X as discovered)_` placeholders — the template is genuinely useful; the data is not.
- Anything else that references the maintainer, their org, their workspace, or their local filesystem.

This list is not exhaustive. If you add a new ID or org-specific reference to the local SKILL.md, add a matching entry here before the next mirror.

## Verification

Before committing a sanitized file, run:

```bash
grep -nE "9002212861|Habuild|Himanshu|100817722|901613959985|Sketch Roadmap|/Users/hkalra|ops@canvasx" skills/*/SKILL.md
```

Must return zero matches. If anything hits, scrub before commit.

(Extend the regex with new identifiers as the maintainer's environment evolves.)

## CLI-drift awareness

SKILL.md docs are tightly coupled to `canvas-cli.js`. When the CLI bundle changes:

- Pull the latest `canvas-cli.js` locally (`cp ~/.sketch/skills-repo/skills/canvas/canvas-cli.js ~/.claude/skills/canvas/canvas-cli.js` after a `git pull` inside the skills-repo cache).
- Sanity-check the subcommands referenced in every SKILL.md against `node canvas-cli.js --help`. Flag unknown-command failures in a follow-up PR.
- Recent rename examples (keep an eye out for more):
  - `get-app-components` + `bulk-get-components` → merged into `get-components --apps <slug-or-csv>`
  - `clickup-get-tasks` + `clickup-get-tasks-by-user` + `clickup-get-tasks-in-space` → merged into `clickup-find-tasks`
  - Added: `clickup-context`, `clickup-project-summary`

## Related context

- Canonical local skill paths: `~/.claude/skills/canvas/SKILL.md`, `~/.claude/skills/clickup/SKILL.md`, `~/.claude/skills/icp-discovery/SKILL.md`.
- Sketch's sync code: `packages/server/src/skills/sync.ts` in the `canvasxai/sketch` repo.
- SDK Bash-tool pipe block and the `sh -c` wrapper workaround: `.planning/CANVAS_WRAPPER_AGENT_SELF_DEBUG.md` and `.planning/WORKFLOW-SESSION-HANDOFF.md` in the `sketch` repo.
