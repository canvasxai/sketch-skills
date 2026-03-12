---
name: canvas
description: Interact with Canvas AI -- search apps, discover components, execute actions, web search, web scrape, and API calls via the Pipedream platform. Use when the user asks about Canvas, app integrations, automation, Pipedream actions, or wants to search/scrape the web or call APIs.
provider-type: canvas
---

# Canvas

Canvas is an automation and integration platform. This skill provides access to app discovery, component lookup, action execution, web search/scrape, API calls, and Twitter data retrieval.

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

### Action Execution

**direct-execute-action**
Before calling:

1. Use `get-component-definition` to get the input schema
2. Use `get-accounts` to verify the user has a connected account
3. Pass auth as `{ "app_slug": { "authProvisionId": "<token>" } }`

```bash
CANVAS_API_KEY_MCP=$CANVAS_API_KEY CANVAS_USER_EMAIL=$USER_EMAIL node ~/.claude/skills/canvas/canvas-cli.js direct-execute-action --component-key slack_slack-send-message --configured-props '{"channel":"#general","text":"Hello!","slack":{"authProvisionId":"apn_xxxx"}}' --output json
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

**direct-execute-api-call**

```bash
CANVAS_API_KEY_MCP=$CANVAS_API_KEY CANVAS_USER_EMAIL=$USER_EMAIL node ~/.claude/skills/canvas/canvas-cli.js direct-execute-api-call --url "https://api.example.com/data" --method GET --output json
```

### Twitter (no connected account required)

**direct-execute-twitter-get-tweets**

```bash
CANVAS_API_KEY_MCP=$CANVAS_API_KEY CANVAS_USER_EMAIL=$USER_EMAIL node ~/.claude/skills/canvas/canvas-cli.js direct-execute-twitter-get-tweets --query "AI agents" --limit 5 --output json
```

**direct-execute-twitter-get-profiles**

```bash
CANVAS_API_KEY_MCP=$CANVAS_API_KEY CANVAS_USER_EMAIL=$USER_EMAIL node ~/.claude/skills/canvas/canvas-cli.js direct-execute-twitter-get-profiles --screen-name elonmusk --output json
```

**direct-execute-twitter-get-recent-posts**

```bash
CANVAS_API_KEY_MCP=$CANVAS_API_KEY CANVAS_USER_EMAIL=$USER_EMAIL node ~/.claude/skills/canvas/canvas-cli.js direct-execute-twitter-get-recent-posts --screen-name elonmusk --output json
```

**direct-execute-twitter-get-user-id**

```bash
CANVAS_API_KEY_MCP=$CANVAS_API_KEY CANVAS_USER_EMAIL=$USER_EMAIL node ~/.claude/skills/canvas/canvas-cli.js direct-execute-twitter-get-user-id --screen-name elonmusk --output json
```

**direct-execute-twitter-get-multiple-profiles**

```bash
CANVAS_API_KEY_MCP=$CANVAS_API_KEY CANVAS_USER_EMAIL=$USER_EMAIL node ~/.claude/skills/canvas/canvas-cli.js direct-execute-twitter-get-multiple-profiles --user-ids "123,456" --output json
```

## Workflow Example

1. User: "Find me Slack integrations on Canvas"
2. Search apps: `search-apps --queries "slack"`
3. Get components: `get-app-components --app slack --component-type action`
4. Get definition: `get-component-definition --key slack_slack-send-message`
5. Check accounts: `get-accounts --app slack`
6. Execute: `direct-execute-action --component-key slack_slack-send-message --configured-props '{...}'`

## Error Handling

- Non-zero exit code: show the error message to the user
- 401/403 in output: tell user the API key may be invalid, ask an admin to check the Canvas integration provider config in the dashboard
- Missing connected account: guide user to set up accounts in Canvas (app.canvasx.ai) first
