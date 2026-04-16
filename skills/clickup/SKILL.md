---
name: clickup
description: ClickUp is the team's primary project management tool. Use this skill for all task and project management operations — fetching tasks, creating issues, updating status, checking priorities, listing work by user or space, and managing the team's backlog. Trigger whenever the user asks about tasks, tickets, project status, assignments, or anything tracked in ClickUp.
provider-type: canvas
---

# ClickUp

ClickUp is the team's _primary project management tool_. Use this skill for all task, project, and team management operations — creating tasks, checking status, updating priorities, listing work, and more.

This skill runs via the Canvas CLI (`$CANVAS_CLI`). Auth is injected automatically — never pass `accountId` or credentials manually.

## Workspace Resolution

On first use, resolve the workspace and cache the `team_id` for subsequent calls:

```bash
$CANVAS_CLI fetch-remote-options \
  --component-key clickup-get-tasks-by-user \
  --field-name team_id \
  --output json
```

Similarly, resolve `space_id`, `list_id`, `folder_id`, and `user_id` via `fetch-remote-options` as needed — then reuse them for the rest of the conversation. Don't ask the user for raw IDs unless they volunteer them.

## Dos and Don'ts

### DO

- Run all `$CANVAS_CLI` commands directly in Bash — no subagent needed.
- Use `--current-configuration '{"key":"value"}'` (single-quoted JSON) when calling `fetch-remote-options` with a dependent dropdown.
- Use `archived: false` and `page: 0` on list/task fetches to keep response size manageable.
- Cache resolved IDs (workspace, space, list, user) within the conversation to avoid redundant lookups.
- Use `--search-query "<name>"` with `fetch-remote-options` to narrow results when looking up a specific user, space, or list by name.
- **If results are ambiguous** (multiple matches returned), stop and present the options to the user — ask them to confirm which one they mean before proceeding. Never guess or pick the first result.
- Prefer bounding results via action parameters (`page`, `archived`, `limit`) instead of shell post-processing. If you need shell post-processing (`jq`, `grep`, `wc`), wrap the CLI invocation in `sh -c` first:
  ```bash
  sh -c '"$CANVAS_CLI" <subcommand> [flags] --output json' | jq ...
  ```

### DON'T

- **Never** pass `"accountId"` in `--configured-props` — auth is injected automatically by the CLI.
- **Never** pipe `$CANVAS_CLI ...` directly — the SDK Bash tool returns `(eval):1: permission denied:` when a command from an env var appears on the left side of a pipe. Always use the `sh -c` wrapper form shown above.
- **Never** hardcode IDs — always resolve them via `fetch-remote-options` on first use.

## Available Actions

### Tasks

| Key                          | Description                                                    |
| ---------------------------- | -------------------------------------------------------------- |
| `clickup-create-task`        | Create a new task in a ClickUp list                            |
| `clickup-get-task`           | Retrieve a single task by its ID                               |
| `clickup-update-task`        | Update an existing task's properties                           |
| `clickup-delete-task`        | Delete a task from the workspace                               |
| `clickup-get-tasks`          | Retrieve all tasks from a list                                 |
| `clickup-get-tasks-by-user`  | Get tasks assigned to a specific user                          |
| `clickup-get-tasks-in-space` | Get all tasks in a space (across all folders and lists)         |
| `clickup-get-users`          | Get team members/users from the workspace                      |

### Comments

| Key                           | Description                                                |
| ----------------------------- | ---------------------------------------------------------- |
| `clickup-create-task-comment` | Add a comment to a task (optionally notify or tag users)   |
| `clickup-get-task-comments`   | Retrieve all comments from a task                          |
| `clickup-create-list-comment` | Add a comment to a list                                    |
| `clickup-get-list-comments`   | Retrieve all comments from a list                          |
| `clickup-update-comment`      | Update a comment's text or mark it resolved/unresolved     |
| `clickup-delete-comment`      | Delete a comment from a task or list (irreversible)        |

### Docs

| Key                       | Description                                          |
| ------------------------- | ---------------------------------------------------- |
| `clickup-search-docs`     | Search for docs by name or keyword                   |
| `clickup-create-doc`      | Create a new doc (optionally with an initial page)   |
| `clickup-get-doc`         | Retrieve a doc's metadata including its page IDs     |
| `clickup-get-doc-page`    | Retrieve the content of a single page from a doc     |
| `clickup-create-doc-page` | Create a new page inside a doc (optionally nested)   |
| `clickup-edit-doc-page`   | Update a page's name and/or content                  |

## Common Workflows

### Fetch tasks for a user

First resolve the user ID:

```bash
$CANVAS_CLI fetch-remote-options \
  --component-key clickup-get-tasks-by-user \
  --field-name assignee_id \
  --current-configuration '{"team_id":"<team_id>"}' \
  --search-query "<name>" \
  --output json
```

Then fetch tasks:

```bash
$CANVAS_CLI direct-execute-action \
  --component-key clickup-get-tasks-by-user \
  --configured-props '{"team_id":"<team_id>","assignee_id":"<user_id>","archived":false,"page":0}' \
  --output json
```

### Get all tasks in a list

```bash
$CANVAS_CLI direct-execute-action \
  --component-key clickup-get-tasks \
  --configured-props '{"list_id":"<list_id>"}' \
  --output json
```

### Create a task

```bash
$CANVAS_CLI direct-execute-action \
  --component-key clickup-create-task \
  --configured-props '{"list_id":"<list_id>","name":"<task name>","description":"<desc>","assignees":["<user_id>"],"priority":2}' \
  --output json
```

Priority values: `1` = Urgent, `2` = High, `3` = Normal, `4` = Low

### Update a task

```bash
$CANVAS_CLI direct-execute-action \
  --component-key clickup-update-task \
  --configured-props '{"task_id":"<task_id>","status":"in progress"}' \
  --output json
```

### Get tasks in a space

```bash
$CANVAS_CLI direct-execute-action \
  --component-key clickup-get-tasks-in-space \
  --configured-props '{"team_id":"<team_id>","space_id":"<space_id>"}' \
  --output json
```

### Get comments on a task

```bash
$CANVAS_CLI direct-execute-action \
  --component-key clickup-get-task-comments \
  --configured-props '{"task_id":"<task_id>"}' \
  --output json
```

### Add a comment to a task

```bash
$CANVAS_CLI direct-execute-action \
  --component-key clickup-create-task-comment \
  --configured-props '{"task_id":"<task_id>","comment_text":"<text>","notify_all":false}' \
  --output json
```

### Read a ClickUp doc

Two-pass approach — get metadata first, then fetch page content:

```bash
# Step 1: get doc metadata + page IDs
$CANVAS_CLI direct-execute-action \
  --component-key clickup-get-doc \
  --configured-props '{"team_id":"<team_id>","doc_id":"<doc_id>"}' \
  --output json

# Step 2: fetch page content (use content_format: "text/md" — NOT "markdown")
$CANVAS_CLI direct-execute-action \
  --component-key clickup-get-doc-page \
  --configured-props '{"team_id":"<team_id>","doc_id":"<doc_id>","page_id":"<page_id>","content_format":"text/md"}' \
  --output json
```

> **Note:** `content_format` only accepts `text/md` or `text/plain`. The value `"markdown"` will return a 400 error.

> **Tip:** Doc and workspace IDs are often embedded in the ClickUp URL — e.g. `app.clickup.com/<team_id>/v/dc/<doc_id>/<page_id>`. Extract them directly rather than resolving via `fetch-remote-options`.

### Search docs

```bash
$CANVAS_CLI direct-execute-action \
  --component-key clickup-search-docs \
  --configured-props '{"team_id":"<team_id>","query":"<keyword>"}' \
  --output json
```

## Resolving IDs

ClickUp's hierarchy is: workspace → space → folder → list → task. Resolve each level via `fetch-remote-options`, passing the parent ID in `--current-configuration`.

### Get workspaces

```bash
$CANVAS_CLI fetch-remote-options \
  --component-key clickup-get-tasks-by-user \
  --field-name team_id \
  --output json
```

### Get spaces (requires team_id)

```bash
$CANVAS_CLI fetch-remote-options \
  --component-key clickup-get-tasks-in-space \
  --field-name space_id \
  --current-configuration '{"team_id":"<team_id>"}' \
  --output json
```

### Get lists within a space (requires space_id)

```bash
$CANVAS_CLI fetch-remote-options \
  --component-key clickup-get-tasks \
  --field-name list_id \
  --current-configuration '{"space_id":"<space_id>"}' \
  --output json
```

### Find a user by name

```bash
$CANVAS_CLI fetch-remote-options \
  --component-key clickup-get-tasks-by-user \
  --field-name assignee_id \
  --current-configuration '{"team_id":"<team_id>"}' \
  --search-query "<last name or full name>" \
  --output json
```

> Tip: Search by last name or unique part of the name to avoid ambiguous matches.
