---
name: create-automation
description: Create Sketch automations, recurring tasks, scheduled workflows, and external app event triggers. Always use this skill first when the user asks to "set up an automation", "create an automation", "automate this", "remind me", "run this every day", "monitor this", "whenever/when a new Linear issue is created", "when a ClickUp task is created", or any workflow that should keep running later. Use this before the canvas skill for app-triggered automations; canvas is only the integration backend.
---

# Creating Automations

Use this skill to turn a user request into a Sketch automation. Sketch stores automations in `scheduled_tasks`; some run on a local schedule, and some are triggered by Canvas-managed external app events.

Do not mention Canvas to the user unless you need them to connect an app or resolve an ambiguous setup choice. The user describes apps and events; you choose the right automation shape.

## Surfaces

- `ManageScheduledTasks` creates and manages Sketch automations.
- `$CANVAS_CLI` discovers apps, components, schemas, and Canvas-managed trigger support.
- Native Sketch delivery handles Slack and WhatsApp outputs. Do not create Slack/WhatsApp API actions just to send the final message.

For external app triggers, Canvas is the integration backend and `$CANVAS_CLI` is the discovery surface.
Do not search PATH for `canvas`, `canvas-cli`, or any global binary. Invoke `$CANVAS_CLI ... --output json` directly.

If `$CANVAS_CLI` is unavailable, do not attempt external app triggers. Use a local schedule fallback when that can satisfy the request; otherwise explain that the app-trigger integration is not available.

## Choose The Trigger

### Local schedule

Use a local schedule when the user asks for a time-based run:

- "every morning"
- "every Friday at 5"
- "in 2 hours"
- "daily summary"
- "check every 30 minutes"

Create with `ManageScheduledTasks` using:

- `schedule_type: "cron"` for calendar schedules.
- `schedule_type: "interval"` for fixed second intervals, minimum 60 seconds.
- `schedule_type: "once"` for one-time future runs.
- Leave `timezone` empty unless the user explicitly names a different timezone. The user's default timezone is already supplied by Sketch.

For a single instruction, use the simple form:

```json
{
  "action": "add",
  "prompt": "Summarize today's open Linear issues and send me the highlights.",
  "schedule_type": "cron",
  "schedule_value": "0 9 * * 1-5"
}
```

### Canvas-managed external trigger

Use a Canvas-managed trigger when the user asks for an external app event:

- "when a ClickUp issue is created"
- "whenever a Linear ticket is opened"
- "when a new row appears in Google Sheets"
- "when a GitHub issue gets labeled"

For external app event language like "when" or "whenever" an issue, task, ticket, row, record, or message is created, updated, moved, labeled, or otherwise changes, a Canvas-managed trigger is mandatory when `$CANVAS_CLI` is available. Do not create a scheduled polling fallback unless the user explicitly asks for polling or Canvas trigger discovery/creation is unavailable.

The Sketch workflow still lives in `scheduled_tasks`, but it must be represented as an external Canvas trigger:

- `schedule_type` and `schedule_value` are omitted when calling `ManageScheduledTasks`.
- The trigger step uses `triggerConfig.type: "canvas"`.
- Include `app`, `eventDescription`, and `componentKey` when known.
- Use `status: "pending_canvas_setup"` until a Canvas workflow has actually been created.
- When Canvas returns IDs, update the trigger config with `status: "active"`, `canvasWorkflowId`, `canvasTriggerNodeId`, and `canvasActionNodeId`.

Example pending Sketch workflow:

```json
{
  "action": "add",
  "title": "Handle new ClickUp issues",
  "description": "Runs Sketch when ClickUp creates a new issue.",
  "steps": [
    {
      "id": "trigger",
      "type": "trigger",
      "label": "ClickUp issue created",
      "icon": "clickup",
      "position": { "x": 0, "y": 0 },
      "triggerConfig": {
        "type": "canvas",
        "app": "clickup",
        "eventDescription": "new issue created",
        "componentKey": "clickup.issue.created",
        "status": "pending_canvas_setup"
      }
    },
    {
      "id": "agent1",
      "type": "agent",
      "label": "Handle issue",
      "icon": "sketch-ai",
      "position": { "x": 0, "y": 100 },
      "agentPrompt": "Use the trigger payload to understand the newly created ClickUp issue. Execute the user's requested script or workflow, then report the result clearly."
    }
  ]
}
```

## Canvas Discovery

Before creating a Canvas-managed trigger, discover and verify the app/trigger.

1. Check app connection:

```bash
$CANVAS_CLI search-apps --queries "clickup,linear" --output json
```

If the relevant app is not connected, ask the user to connect it in Settings → Integrations. Do not create a broken trigger workflow.
Do not call the app's own MCP authentication tool as a substitute for Sketch/Canvas connection.

2. Search triggers with `search-components` through the Canvas CLI. If the same surface is exposed as a tool/MCP call, use `search_components`; the intent is the same.

```bash
$CANVAS_CLI search-components --raw '{"queries":[{"app":"clickup","query":"new issue created trigger"}]}' --output json
```

Use `search-components` / `search_components` for trigger discovery.

3. Always inspect the selected trigger component before creating the Sketch workflow:

```bash
$CANVAS_CLI get-component-definition --key <component-key> --output json
```

For Canvas-managed trigger components, read `data.outputSchema` from the response when it is present. Use that schema as the authoring-time contract for the event payload. Canvas-triggered Sketch workflow runs receive that trigger payload at `input.data`; workflow steps should reference fields from that stable path, such as `input.data.task` for task-based triggers. If `data.outputSchema` is absent, do not guess internal Canvas wrapper paths like `input.output.data`, `input.data.data`, or raw node-output structures.

4. Resolve required dropdowns with remote options only when needed:

```bash
$CANVAS_CLI fetch-remote-options --component-key <component-key> --field-name <field> --output json
```

If multiple options match, ask the user. Do not guess.

## Creating The Canvas Workflow

When Canvas exposes workflow creation, create the Canvas-side two-node workflow after the Sketch workflow exists. This should create and deploy the Canvas workflow by default:

1. Canvas trigger node: the selected external app trigger component.
2. Sketch action node: call the Sketch workflow action with the Sketch workflow ID.

Use the Canvas command/tool intended for this purpose. The expected tool name is `create_sketch_trigger_workflow`; if surfaced as a CLI command, it is exposed as `create-sketch-trigger-workflow`.

```bash
$CANVAS_CLI create-sketch-trigger-workflow \
  --sketch-workflow-id <scheduled_task_id> \
  --trigger-app-slug <app-slug> \
  --trigger-component-key <component-key> \
  --trigger-configured-props '<json>' \
  --output json
```

After creation, update the Sketch automation with Canvas IDs. Mark the trigger active only when Canvas returns `isLive: true` or `deploymentStatus: "deployed"`. If Canvas returns a draft or pending deployment state, keep the trigger `pending_canvas_setup` and tell the user the Sketch workflow is staged but the Canvas trigger is not live yet.

```json
{
  "action": "update",
  "task_id": "<scheduled_task_id>",
  "steps": [
    {
      "id": "trigger",
      "type": "trigger",
      "label": "ClickUp issue created",
      "icon": "clickup",
      "position": { "x": 0, "y": 0 },
      "triggerConfig": {
        "type": "canvas",
        "app": "clickup",
        "eventDescription": "new issue created",
        "componentKey": "clickup.issue.created",
        "configuredProps": {},
        "status": "active",
        "canvasWorkflowId": "<canvas_workflow_id>",
        "canvasTriggerNodeId": "<canvas_trigger_node_id>",
        "canvasActionNodeId": "<canvas_action_node_id>"
      }
    },
    {
      "id": "agent1",
      "type": "agent",
      "label": "Handle issue",
      "icon": "sketch-ai",
      "position": { "x": 0, "y": 100 },
      "agentPrompt": "<same prompt as creation>"
    }
  ]
}
```

If Canvas workflow creation is not available yet, still create the Sketch workflow with `pending_canvas_setup` so it is visible in the Automations UI as Canvas-managed pending setup. Tell the user the Sketch workflow is staged and Canvas trigger wiring is pending.

## Workflow Step Guidance

Do not collapse compound work into a single agent step. If the user asks for a workflow that reads app data, reasons over it, writes app data, and reports a result, create distinct execution steps for those phases. A Canvas trigger only decides when the workflow starts; it does not replace the Sketch workflow steps that should run after the event arrives.

Use multiple non-trigger steps when the request contains more than one meaningful operation, especially any combination of:

- extracting or normalizing an event payload
- fetching extra app data
- reasoning, classification, planning, summarization, or generation
- creating, updating, or searching external app records
- sending or returning a final user-facing summary

Prefer action steps for deterministic app reads/writes and bounded data transforms. Prefer agent steps for interpretation, planning, generation, and final prose. Keep each step small enough that its output is a useful input to the next step.

### Trigger step

Every multi-step workflow starts with one trigger step.

- Scheduled trigger: `triggerConfig: { "type": "schedule" }`.
- Canvas trigger: `triggerConfig: { "type": "canvas", ... }`.

### Agent step

Use agent steps for reasoning, summarizing, transforming, deciding, and executing Sketch-native behavior. Write the prompt so it can run without the chat history.

Good agent prompts include:

- What input to expect.
- What action to take.
- What output to send or record.
- Any constraints from the user.

### Action step

Use action steps when the workflow needs deterministic JavaScript glue: small JSON transforms, bounded loops, and Canvas app reads/writes whose component key and props are known.

Action scripts run as an async function body with this signature:

```js
async function action(input, ctx, signal) {
  // your script body
}
```

The script body should `return` a small JSON-serializable value. `input` is the previous step output or trigger payload. `ctx` contains:

- `ctx.env.CANVAS_CLI` — brokered Canvas CLI launcher for action execution.
- `ctx.env.INTEGRATION_CLI` — temporary legacy alias for `ctx.env.CANVAS_CLI`; prefer `CANVAS_CLI`.
- `ctx.workspaceDir` — workflow workspace directory.
- `ctx.log` — step-scoped logger.

Use `ctx.env`, not `process.env`. When invoking Canvas from a script, pass the script env explicitly:

```js
const { execFile } = await import("node:child_process");
const { promisify } = await import("node:util");
const execFileAsync = promisify(execFile);

const { stdout } = await execFileAsync(
  ctx.env.CANVAS_CLI,
  [
    "direct-execute-action",
    "--component-key",
    "gmail-find-email",
    "--configured-props",
    JSON.stringify({ query: "newer_than:1h" }),
    "--output",
    "json"
  ],
  { cwd: ctx.workspaceDir, env: { ...process.env, ...ctx.env }, encoding: "utf8" }
);

const result = JSON.parse(stdout);
return {
  count: result.data?.messages?.length ?? 0,
  messages: (result.data?.messages ?? []).map((m) => ({
    id: m.id,
    from: m.from,
    subject: m.subject,
    date: m.date,
    snippet: m.snippet
  }))
};
```

Do not depend on stdin/stdout for step output. Do not return raw app responses; project them into the smallest useful object before returning. Avoid long-running CPU loops, background processes, and `process.exit()`.

Prefer Canvas-managed triggers plus agent steps when the app event itself should trigger the workflow. For app reads/writes that must run during the workflow, first confirm the current runtime supports the operation through Canvas. If it does not, tell the user the automation can be staged but the app action cannot be safely executed yet.

## Verification

### Before creation

- For schedule workflows, confirm the cron/interval/once value matches the user's requested timing.
- For Canvas triggers, confirm app connection, selected trigger component, and required configured props.
- If the external event has ambiguous scope, ask before creating. Examples: which ClickUp workspace/list, which Linear team/project, which GitHub repo.

### After creation

For local schedules:

1. Use `ManageScheduledTasks { "action": "run", "task_id": "<id>" }` when a manual run is meaningful.
2. Use `ManageScheduledTasks { "action": "getRun", "task_id": "<id>" }` to inspect the result.
3. Report whether it ran with non-empty useful output.

For Canvas triggers:

- Do not use "Run now" as proof that the external trigger is wired.
- Confirm the Sketch workflow exists and appears as Canvas-managed.
- If Canvas workflow creation ran, report the Canvas workflow ID and active status.
- If it is pending, say it is staged in Sketch and waiting for Canvas trigger wiring.

## Error Handling

- App not connected: create the Sketch workflow as `pending_canvas_setup` when the app and trigger component are identifiable, then tell the user it is staged and will activate after the app is connected in Settings → Integrations.
- No matching trigger component: say the trigger is not available yet and offer a scheduled fallback.
- Ambiguous trigger configuration: ask a focused question with the viable options.
- Canvas workflow creation unavailable: create the Sketch workflow as `pending_canvas_setup` and explain that trigger wiring is pending.
- Canvas creation failed after Sketch workflow creation: update the trigger config to `status: "error"` with a short `errorMessage` when possible.

## Output To User

Keep the response short:

- What was created.
- Trigger type: scheduled or external app event.
- Current status: active, pending Canvas setup, or error.
- Any manual step the user must take, such as connecting an app.
