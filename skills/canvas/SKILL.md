---
name: canvas
description: Interact with Canvas AI -- search apps, discover components, execute actions, web search, web scrape, and automation via the Pipedream platform and Canvas custom integrations. Use when the user asks about Canvas, app integrations, automation, Pipedream actions, or wants to search/scrape the web.
provider-type: canvas
---

# Canvas

Canvas is an automation and integration platform. This skill provides access to app discovery, component lookup, action execution (Pipedream and Canvas custom), and web search/scrape.

## Authentication

Before any Canvas operation, call the `getProviderConfig` tool to get the integration provider credentials.

If it returns `configured: false`, tell the user that Canvas is not configured and an admin needs to set up the Canvas integration provider in the Sketch dashboard.

The user's email is available from your context (system prompt in DMs, or the conversation messages in group chats). Use it as CANVAS_USER_EMAIL.

NEVER echo or log the API key value in responses.

## CLI Usage

All Canvas operations go through the CLI. Invocation pattern:

```bash
CANVAS_API_KEY_MCP=<apiKey from getProviderConfig> CANVAS_USER_EMAIL=<user email> node ~/.claude/skills/canvas/canvas-cli.js <subcommand> [flags] --output json
```

Always use `--output json` for structured results you can parse and present clearly.

## Available Subcommands

### App Discovery

**search-apps**

```bash
CANVAS_API_KEY_MCP=$CANVAS_API_KEY CANVAS_USER_EMAIL=$USER_EMAIL node ~/.claude/skills/canvas/canvas-cli.js search-apps --queries "slack,gmail,notion" --output json
```

### Component Discovery

**search-components**

```bash
CANVAS_API_KEY_MCP=$CANVAS_API_KEY CANVAS_USER_EMAIL=$USER_EMAIL node ~/.claude/skills/canvas/canvas-cli.js search-components --raw '{"queries": [{"app": "slack", "query": "send message"}]}' --output json
```

**get-app-components**

```bash
CANVAS_API_KEY_MCP=$CANVAS_API_KEY CANVAS_USER_EMAIL=$USER_EMAIL node ~/.claude/skills/canvas/canvas-cli.js get-app-components --app slack --component-type action --output json
```

**bulk-get-components**

```bash
CANVAS_API_KEY_MCP=$CANVAS_API_KEY CANVAS_USER_EMAIL=$USER_EMAIL node ~/.claude/skills/canvas/canvas-cli.js bulk-get-components --apps "slack,gmail" --component-type action --output json
```

**get-component-definition**

```bash
CANVAS_API_KEY_MCP=$CANVAS_API_KEY CANVAS_USER_EMAIL=$USER_EMAIL node ~/.claude/skills/canvas/canvas-cli.js get-component-definition --key slack_slack-send-message --output json
```

### Accounts

**get-accounts**

```bash
CANVAS_API_KEY_MCP=$CANVAS_API_KEY CANVAS_USER_EMAIL=$USER_EMAIL node ~/.claude/skills/canvas/canvas-cli.js get-accounts --app slack --output json
```

### Pipedream Action Execution

**direct-execute-action**

For Pipedream-hosted actions (e.g. Slack, Gmail, Notion). Before calling:

1. Use `get-component-definition` to get the input schema
2. Use `get-accounts` to verify the user has a connected account
3. Pass auth as `{ "app_slug": { "authProvisionId": "<token>" } }`

```bash
CANVAS_API_KEY_MCP=$CANVAS_API_KEY CANVAS_USER_EMAIL=$USER_EMAIL node ~/.claude/skills/canvas/canvas-cli.js direct-execute-action --component-key slack_slack-send-message --configured-props '{"channel":"#general","text":"Hello!","slack":{"authProvisionId":"apn_xxxx"}}' --output json
```

### Canvas Action Execution

**direct-execute-canvas-action**

Unified executor for Canvas custom integrations. Supports: **Fireflies, Aimfox, GitHub, ClickUp, Twitter, LinkedIn**. Each app resolves credentials automatically — per-user secrets for Fireflies/Aimfox/GitHub/ClickUp, system-level credentials for Twitter/LinkedIn.

Before calling:

1. Use `search-components` or `get-app-components` to discover available action keys for the app
2. Use `get-component-definition` to get the input schema for the action
3. Pass `--component-key` with the action key and `--configured-props` matching the schema

```bash
CANVAS_API_KEY_MCP=$CANVAS_API_KEY CANVAS_USER_EMAIL=$USER_EMAIL node ~/.claude/skills/canvas/canvas-cli.js direct-execute-canvas-action --component-key <action-key> --configured-props '{"..."}' --output json
```

**Credential models by app:**

| App | Credentials | User setup required? |
|-----|-------------|---------------------|
| Fireflies | Per-user API key via Settings → Secrets | Yes |
| Aimfox | Per-user API key via Settings → Secrets | Yes |
| GitHub | Per-user PAT via Settings → Secrets | Yes |
| ClickUp | OAuth token via Settings → Secrets | Yes |
| Twitter | System-level (env var) | No |
| LinkedIn | System-level (env var) | No |

For apps requiring per-user secrets, use `user_secrets_get_status` to check if the user has configured their credentials before attempting execution. If secrets are missing, guide the user to Settings → Secrets to connect the app.

**Examples:**

Twitter — search tweets (system credentials, no user setup):
```bash
CANVAS_API_KEY_MCP=$CANVAS_API_KEY CANVAS_USER_EMAIL=$USER_EMAIL node ~/.claude/skills/canvas/canvas-cli.js direct-execute-canvas-action --component-key twitter-get-tweets --configured-props '{"query":"AI agents","limit":5}' --output json
```

Fireflies — list transcripts (per-user secret required):
```bash
CANVAS_API_KEY_MCP=$CANVAS_API_KEY CANVAS_USER_EMAIL=$USER_EMAIL node ~/.claude/skills/canvas/canvas-cli.js direct-execute-canvas-action --component-key fireflies-list-transcripts --configured-props '{}' --output json
```

### Web Tools (no auth required)

**direct-execute-web-search**

```bash
CANVAS_API_KEY_MCP=$CANVAS_API_KEY CANVAS_USER_EMAIL=$USER_EMAIL node ~/.claude/skills/canvas/canvas-cli.js direct-execute-web-search --query "TypeScript best practices" --limit 5 --output json
```

**direct-execute-web-scrape**

```bash
CANVAS_API_KEY_MCP=$CANVAS_API_KEY CANVAS_USER_EMAIL=$USER_EMAIL node ~/.claude/skills/canvas/canvas-cli.js direct-execute-web-scrape --url "https://example.com" --formats markdown --only-main-content true --output json
```

## Workflow Examples

### Pipedream action (e.g. Slack)

1. User: "Send a message to #general on Slack"
2. Search apps: `search-apps --queries "slack"`
3. Get components: `get-app-components --app slack --component-type action`
4. Get definition: `get-component-definition --key slack_slack-send-message`
5. Check accounts: `get-accounts --app slack`
6. Execute: `direct-execute-action --component-key slack_slack-send-message --configured-props '{...}'`

### Canvas action (e.g. Fireflies)

1. User: "Get my recent Fireflies transcripts"
2. Search components: `search-components --raw '{"queries": [{"app": "fireflies", "query": "list transcripts"}]}'`
3. Get definition: `get-component-definition --key fireflies-list-transcripts`
4. Execute: `direct-execute-canvas-action --component-key fireflies-list-transcripts --configured-props '{}'`

## Error Handling

- Non-zero exit code: show the error message to the user
- 401/403 in output: tell user the API key may be invalid, ask an admin to check the Canvas integration provider config in the dashboard
- `MISSING_CREDENTIALS` error: use `user_secrets_get_status` to confirm, then guide user to connect the app via Settings → Secrets
- `INVALID_KEY` error: the action key is not recognized — use component discovery to find valid keys
- Missing connected account (Pipedream actions): guide user to set up accounts in Sketch dashboard