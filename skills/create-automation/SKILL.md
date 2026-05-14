---
name: create-automation
description: >
  Set up workflows, automations, and recurring tasks involving integrations or
  multi-step pipelines. Use this skill whenever the user wants to automate
  something scheduled or multi-step that touches connected apps, with delivery
  to Slack or WhatsApp. Triggers: "remind me every morning to…", "every Monday
  send me…", "summarize my X and message me", "set up an automation for…",
  "create a workflow that…", "schedule a daily digest of…", "fetch X from app
  A and post to app B", "build me a recurring task", "automate this".
  Use even when the user doesn't say "automation" — if the ask is recurring or
  multi-step across apps, that's this skill. Do NOT use for one-off "do X right
  now" tasks — handle those directly with the relevant integration without
  building a workflow.
---

# Creating Automations

## Native delivery (no integration needed)

This agent runs on **Slack** and **WhatsApp** and natively delivers messages to either platform — no third-party messaging integration required. For "notify me / remind me / summarize and send" tasks, use a workflow whose final step is an `agent` step and set `output_platform` to `"slack"` or `"whatsapp"`. The agent's text output is delivered directly. Never wire up a Slack/WhatsApp API component for messaging.

## Before you start

1. `$CANVAS_CLI search-apps --output json` (no queries) — lists apps the user has connected. Each item has `nameSlug`, `name`, `isConnected`. Canvas is always installed; don't pre-check `getProviderConfig`.
2. If the user asks for an app they haven't connected, tell them to connect it in Settings → Integrations. Don't build a broken workflow.

## Simple vs multi-step

- **Simple**: `ManageScheduledTasks { action: "add", prompt, schedule_type, schedule_value }`. Use for "remind me…" or any single-prompt instruction.
- **Multi-step**: `ManageScheduledTasks { action: "add", title, steps, schedule_type, schedule_value }`. Use when the task fetches from one app, processes, and/or sends to another.

**Timezone**: leave the `timezone` field empty. The user's tz from the `<time>` block in your system prompt is used as the default automatically. Only set `timezone` when the user explicitly names a different one for the task ("schedule this in PST"). If the user says "set my timezone to X" instead, call `SetUserTimezone({ timezone: "X" })` — that updates their default for everything, not just one task.

## Step types

- `trigger`: first step, `triggerConfig: { type: "schedule" }`.
- `action`: Node.js script in a sandbox; calls connected apps via the integration CLI. Use for fetching, creating, sending, reading.
- `agent`: AI step; no integration needed. Use for summarize, analyze, transform, decide.

## Action-step scripts

Input from the previous step is available as `input` (parsed JSON). Return a value from the wrapper — it becomes the next step's input.

### Runtime env contract

The action-step child process gets a **locked-down env** — `process.env` is not inherited. Exactly these vars are set:

- `INTEGRATION_CLI` — absolute path to the CLI's `.js` file. Always invoke as `node $INTEGRATION_CLI <subcommand> …`. Never hardcode a path. If unset, you should not be using an action step.
- Provider credentials (e.g. `CANVAS_API_KEY_MCP`, `CANVAS_USER_EMAIL` for the Canvas provider) — set automatically. Treat as opaque; never read or log.
- `PATH=/usr/bin:/bin:/usr/local/bin:/opt/homebrew/bin` (fixed).
- `NODE_NO_WARNINGS=1`.

Not present in the action sandbox: `HOME`, chat-session env, `ANTHROPIC_API_KEY`, `$CANVAS_CLI` (chat-only — see below). Don't shell out to anything that needs `$HOME` (caches, configs).

In **chat mode** (used only for sampling per "Verifying before commit"), `$CANVAS_CLI` is a shell-script launcher that goes through a credentialed broker — invoke it directly (no `node` prefix). It runs the same underlying `canvas-cli.js` as `$INTEGRATION_CLI`, so response shapes are identical.

### Auth model

The CLI resolves auth server-side from the user's connected accounts. **`configuredProps` carries only the action's input fields — no `authProvisionId`, no `accountId`.** If the user isn't connected, the call returns `CONNECTION_NOT_CONNECTED`.

### Script shape

```javascript
const { execSync } = require("child_process");

const props = JSON.stringify({ maxResults: 10 });
const raw = execSync(
  `node $INTEGRATION_CLI direct-execute-action --component-key gmail-find-email --configured-props '${props}' --output json`,
  { env: process.env, encoding: "utf8", shell: "/bin/sh" },
);
const parsed = JSON.parse(raw);

// Project down — never return the raw blob (see "Slim the output").
const messages = (parsed.data?.ret ?? []).map((m) => {
  const headers = m.payload?.headers ?? [];
  const h = (name) => headers.find((x) => x.name === name)?.value ?? null;
  return { id: m.id, from: h("From"), subject: h("Subject"), date: h("Date"), snippet: m.snippet };
});
return { count: messages.length, messages };
```

Rules:
- Always `node $INTEGRATION_CLI …`.
- Pass `env: process.env` to `execSync` so credential env vars propagate.
- Always `return` from the wrapper — that becomes the next step's input.
- Default timeout: 30 minutes (`step.timeout` overrides, in seconds).

### Slim the output

The runtime passes the **whole return value** to the next step — no projection. For an `agent` downstream, fat payloads (MIME parts, base64, HTML bodies, label arrays) inflate the prompt every run, waste tokens, and bury the signal. For another `action` downstream, they bloat run history.

Rules of thumb:
- Drop envelopes/metadata (`success`, `executedAt`, paging) unless the next step reads them.
- For lists, `.map()` items into a minimal object — keep `id`, key display fields, and a snippet/summary; drop full bodies.
- Bound the CLI request itself first (`maxResults`, date filters, `metadataOnly: true` where supported) — slim-at-fetch beats slim-at-return.
- Don't trust training-data API shapes — Canvas components flatten/rename fields. `get-component-definition` documents only the **input** schema, not the output. Sample first (next section).

## Verifying before commit (MANDATORY)

Every action step must be invoked against live data **before** `ManageScheduledTasks { action: "add" }`. Empty-result runs give false confidence: a workflow can complete green with zero items while the projection is silently broken against a shape that never existed.

### Read-only actions

1. **Sample via `$CANVAS_CLI`** (chat-mode launcher; runs the same `canvas-cli.js` the action sandbox does, so the response shape is identical). Run the command bare — the Bash tool captures stdout into the tool result for you to read directly.

   ```bash
   $CANVAS_CLI direct-execute-action --component-key <key> --configured-props '{"limit":3}' --output json
   ```

   Use a **broad** query first (`limit: 3`, no filters) — empty responses teach you nothing about field shapes. If you need to filter/transform with `jq`, wrap in `sh -c` — a bare `$CANVAS_CLI ... | jq` fails under the SDK Bash tool with `(eval):1: permission denied:` (env-var command on the left of a pipe):
   ```bash
   sh -c '"$CANVAS_CLI" direct-execute-action --component-key <key> --configured-props '"'"'{"limit":3}'"'"' --output json' | jq '.data.ret[0] | keys'
   ```

2. **Note the shape.** From the tool result, capture top-level keys and, for lists, the keys of the first item.

3. **Field contract check.** For each downstream step, list every field its prompt or script reads and confirm it's present and non-empty. Example:
   ```
   fetch_meetings real shape: data.transcripts[0] keys → [id, title, url, createdAt, durationMs]

   Downstream (process_todos prompt):
     id              ✓
     title           ✓
     date            ✗ (response has createdAt)
     participants[]  ✗ (not on list endpoint)
     action_items    ✗ (nowhere in component)
   → Cannot build as requested. Surface to user.
   ```
   Optional chaining and `?? []` will silently mask missing fields as empty arrays — that's exactly the failure mode this check exists to catch.

4. Only after every action step passes, call `add`.

### Agent steps

Take the projected sample from the upstream action. List every field the agent prompt references; confirm each is present and non-empty in the sample. If a referenced field is missing or always empty, fix the prompt or the upstream projection before commit.

### Write/destructive actions (create/send/update/delete)

Can't be sampled live without side effects. Instead:
- Confirm the input schema via `get-component-definition`.
- Show the user the exact payload (all values filled in) and confirm before commit.
- You still must pre-commit-verify the upstream **read** actions — otherwise the payload is built from a broken projection at runtime.

## After creation (secondary check)

If pre-commit verification was done properly, this just confirms scheduler wiring.

1. `ManageScheduledTasks { action: "run", task_id }`.
2. `action: "getRun"` to inspect step outputs.
3. Fix with `action: "updateStepContent"` if needed.
4. Only report "working" if the run produced **non-empty** data. If it completed but returned zero items, say so explicitly.

## Discovering components

- `$CANVAS_CLI search-apps --output json` — list connected apps (no args) or search the catalog (`--queries`).
- `$CANVAS_CLI get-components --apps <slug> --component-type action` — list actions for an app. Slugs use underscores (`google_sheets`); always `--apps`, never `--app`.
- `$CANVAS_CLI search-components --raw '{"queries":[{"app":"<slug>","query":"<intent>"}]}'` — natural-language discovery within an app.
- `$CANVAS_CLI get-component-definition --key <component-key>` — input schema for a specific component. (Note: this subcommand uses `--key`; `direct-execute-action` and `fetch-remote-options` use `--component-key`.)
- `$CANVAS_CLI fetch-remote-options --component-key <key> --field-name <field> [--search-query <q>]` — resolve dropdown values (channel name → ID, list name → ID). Use only when you don't already have the ID. If multiple matches, surface to the user — don't guess.

## Common patterns

- **Fetch → Summarize → Deliver**: action → agent → native delivery via `output_platform`.
- **Monitor → Alert**: action (check) → agent (decide if alert-worthy) → output if yes.
- **Collect → Transform → Store**: action (fetch) → agent (restructure) → action (write to sheet/task).
