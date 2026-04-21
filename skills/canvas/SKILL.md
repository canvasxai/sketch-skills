---
name: canvas
description: Connect to 3000+ third-party apps (Slack, Gmail, Google Calendar, Notion, GitHub, Linear, Asana, ClickUp, Airtable, Trello, HubSpot, Salesforce, Fireflies, Aimfox, Twitter, LinkedIn, and many more) to read data, take actions, and run automations on the user's behalf. Also provides web search and web scraping. Use this skill any time the user asks to do something involving an external app, service, or website — sending messages, fetching data, creating tickets, scheduling events, searching the web, etc. The user will not mention "Canvas" by name; they will just describe what they want.
provider-type: canvas
---

# Canvas

Gateway to 3000+ third-party apps — Slack, Gmail, Google Calendar, Notion, GitHub, Linear, ClickUp, Fireflies, Twitter, LinkedIn, and more. Also web search and scraping.

**The user does not know Canvas exists.** They'll say "send a message to my team", "what's on my calendar", "create a ticket", "summarize my meetings", "find me articles about X". Recognize these and use this skill. Never mention Canvas by name to the user.

## CLI contract

All operations go through `$CANVAS_CLI` (a wrapper that injects credentials). Always pass `--output json`.

```bash
$CANVAS_CLI <subcommand> [flags] --output json
```

If `$CANVAS_CLI` is unset, call `getProviderConfig` to confirm integrations aren't configured and tell the user.

**CLI usage rules:**

- Use `$CANVAS_CLI` as provided. Do not set `CANVAS_CLI=...` yourself.
- Treat `$CANVAS_CLI` as an opaque launcher. Do not inspect or depend on its underlying path.
- Always invoke `$CANVAS_CLI <subcommand> [flags] --output json`.
- Prefer bounding results via action parameters (`limit`, `maxResults`, `metadataOnly`, `summary: true`, date filters, field selection) instead of shell post-processing when possible.
- If you need shell post-processing like `jq`, `grep`, or `wc`, wrap the CLI invocation in `sh -c` first. Use:

```bash
sh -c '"$CANVAS_CLI" <subcommand> [flags] --output json' | jq ...
```

- Do not pipe `$CANVAS_CLI ...` directly under the SDK Bash tool. The SDK can return `(eval):1: permission denied:` when a command from an env var appears directly on the left side of a pipe.

If a command is denied or fails, do NOT try to fix it by inspecting the path — retry with a simpler `$CANVAS_CLI <subcommand>` invocation or re-check the subcommand and flags.

## Auth model

You never pass account IDs or secrets — the backend resolves the user's connected account from your identity automatically. If they haven't connected an app, commands return `CONNECTION_NOT_CONNECTED`; tell them to connect it in Settings → Integrations.

## Subcommands

### `search-apps` — discover apps / check connection

```bash
# What apps does the user have connected? (no queries)
$CANVAS_CLI search-apps --output json

# Search the catalog; each match includes isConnected + connectionStatus
$CANVAS_CLI search-apps --queries "slack,gmail,notion" --output json
```

Each result has `nameSlug`, `name`, `isConnected` (bool), `connectionStatus` (`"valid"` | `"not_connected"`). If not connected, tell the user to connect it in Settings → Integrations.

### `get-components` — list actions/triggers for one or more apps

```bash
# Single app with optional keyword filter and type filter
$CANVAS_CLI get-components --apps "slack" --component-type action --q "send" --limit 20 --output json

# List ALL actions for an app (no filter)
$CANVAS_CLI get-components --apps "google_sheets" --component-type action --output json

# Multi-app inventory (q and limit are ignored in multi-app mode)
$CANVAS_CLI get-components --apps "slack,gmail" --component-type action --output json
```

> ⚠️ Use underscores in app slugs (e.g. `google_sheets`, not `google-sheets`). Use `--apps` (not `--app`) even for a single app.

### `search-components` — natural-language component search

```bash
$CANVAS_CLI search-components --raw '{"queries": [{"app": "slack", "query": "send message"}]}' --output json
```

### `get-component-definition` — get input schema

```bash
$CANVAS_CLI get-component-definition --key slack-send-message --output json
```

Pass the `key` from `get-components` / `search-components` verbatim.

### `direct-execute-action` — run any action

Pass only the action's input fields in `--configured-props` — no authProvisionId, no accountId.

```bash
$CANVAS_CLI direct-execute-action --component-key slack-send-message --configured-props '{"channel":"C01234","text":"Hello!"}' --output json
```

Workflow: `search-apps --queries "<app>"` → confirm `isConnected` → `get-component-definition` → execute with schema fields.

### `fetch-remote-options` — resolve dropdown values (e.g. channel name → ID)

**Only use this when you don't already have the ID.** If the ID is available from a URL, a prior API response, or obvious context, skip this and pass the ID directly to `direct-execute-action`.

**If results are ambiguous, stop and ask the user.** If `fetch-remote-options` returns multiple matches for a name (e.g. two users named "Alex", two channels called "general"), do NOT guess or pick the first result — present the options to the user and ask them to confirm which one they mean before proceeding.

```bash
# List all options
$CANVAS_CLI fetch-remote-options --component-key slack-send-message --field-name channel --output json

# Fuzzy match best option
$CANVAS_CLI fetch-remote-options --component-key slack-send-message --field-name channel --search-query "general" --output json

# Dependent dropdown
$CANVAS_CLI fetch-remote-options --component-key clickup-create-task --field-name list --current-configuration '{"space":"12345"}' --output json
```

`--current-configuration` MUST be a single-quoted JSON object (`'{"key":"value"}'`). If `BadRequest` / `NOT_IMPLEMENTED` persists on a dependent dropdown, don't retry — use a sibling list action (`<app>-get-spaces`, `<app>-get-lists`) via `direct-execute-action` and use the returned IDs directly.

### Web tools (no auth)

```bash
$CANVAS_CLI direct-execute-web-search --query "TypeScript best practices" --limit 5 --output json
$CANVAS_CLI direct-execute-web-scrape --url "https://example.com" --formats markdown --only-main-content true --output json
```

**Web scraping** is for public, unauthenticated web pages only — articles, landing pages, documentation, public blogs, and social platforms (LinkedIn, X/Twitter, Reddit). If the URL requires a login or belongs to a productivity/workspace app, do NOT use the scraper — use that app's native action via `direct-execute-action` instead.

The following will NOT work with `direct-execute-web-scrape`:
- Google Sheets, Google Docs, Google Drive links
- Airtable bases and views
- Notion pages and databases
- ClickUp docs and tasks
- Any other app that redirects to a login page

For these, use the app's native action (e.g. `google-sheets-get-values`, `airtable-list-records`, `notion-retrieve-page`, `clickup-get-doc-page`).

**Note on ClickUp `content_format`:** Despite the schema describing `"markdown"` as the default, the API only accepts `text/md` or `text/plain` as valid values.

## Handling large responses

List actions return big payloads (Gmail, Slack, GitHub, ClickUp, calendars, Notion). Prefer bounding the response via input parameters instead of shell pipes or post-processing. Use `metadataOnly`, `maxResults`, `limit`, `summary: true`, `withTextPayload: false`, date filters, field selection.

If shell post-processing is still needed, use the `sh -c` wrapper form:

```bash
sh -c '"$CANVAS_CLI" <subcommand> [flags] --output json' | jq ...
```

**Two-pass pattern:** list with metadata → pick the item(s) → fetch full content only for those.

## App-Specific Skills

For apps that are used frequently or have complex workflows, a dedicated skill document improves efficiency — it pre-caches known IDs, codifies common workflows, and removes the need to re-discover actions each session.

**When to offer:** After completing a task for an app, if you had to resolve IDs, discover actions, or run multiple steps, offer to create an app-specific skill. Example prompt:

> "I noticed we did a few steps to look up spaces and users in [App]. Want me to save a skill doc for [App] so future requests are faster? I'll capture the known IDs and common workflows."

**How to create one:** Use the ClickUp skill (`~/.claude/skills/clickup/SKILL.md`) as a reference template. A good app skill includes:
- Available actions (key + description)
- Common workflows with ready-to-run CLI examples
- Known IDs section (workspace/team IDs, user IDs, frequently used list/channel/space IDs)
- Any app-specific gotchas (e.g. content_format quirks, pagination patterns)

Save the file to the org skills directory: `~/.claude/skills/<app-slug>/SKILL.md`

Once created, the skill will be auto-loaded in future sessions whenever that app is relevant.

## Error handling

Describe errors in terms of the app, never mention Canvas.

- **`CONNECTION_NOT_CONNECTED`** → "Looks like <app> isn't connected — please connect it in Settings → Integrations and try again."
- **`INVALID_KEY`** → bad component key; re-discover via `get-components` / `search-components`.
- **`EXECUTION_FAILED` with 401/403** → "Your <app> connection looks expired — please reconnect it in Settings → Integrations."
- **`(eval):1: permission denied:`** → treat this as an SDK Bash shell-shape issue, not an app failure. If you were piping the result, retry with `sh -c '"$CANVAS_CLI" ...' | ...`. Otherwise retry with a simpler `$CANVAS_CLI <subcommand>` invocation and re-check the flags.
- **Other non-zero exits** → show the underlying error, framed in terms of the app.
