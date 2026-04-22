---
name: clickup
description: Work with ClickUp — fetching tasks, creating issues, updating status, checking priorities, listing work by user or space, managing backlogs, and reading or editing docs. Trigger whenever the user asks about anything tracked in ClickUp.
provider-type: canvas
---

# ClickUp

Use this skill for task, project, and team management operations in ClickUp — creating tasks, checking status, updating priorities, listing work, reading or editing docs, and managing comments.

This skill runs via the Canvas CLI (`$CANVAS_CLI`). Auth is injected automatically — never pass `accountId` or credentials manually.

## Org-specific context

At the **start of every session**, read `SKILL-CONTEXT.md` in this directory. It holds the org's known IDs — workspace `team_id`, users, lists, spaces — and lets you skip redundant lookups.

- If `SKILL-CONTEXT.md` has the ID you need, use it directly.
- If an ID is missing, resolve it (see *Resolving IDs* below) and **append it to `SKILL-CONTEXT.md`** so it's available next time. Only track stable entities (workspaces, users, spaces, folders, lists). Never record task IDs or doc IDs there.
- When you run `clickup-context` (or any action that returns a batch of stable entities), record **all** of them in `SKILL-CONTEXT.md` — not just the one you needed for the immediate task. This is the cheapest time to capture them.
- If `SKILL-CONTEXT.md` doesn't exist, start by running `clickup-context` to populate workspace hierarchy, then create the file from the template structure documented inside it.

## Dos and Don'ts

### DO
- Run all `$CANVAS_CLI` commands directly in Bash — no subagent needed.
- Use `--current-configuration '{"key":"value"}'` (single-quoted JSON) when calling `fetch-remote-options` with a dependent dropdown.
- Use `archived: false` and `page: 0` on list/task fetches to keep response size manageable.
- Prefer IDs from `SKILL-CONTEXT.md` to skip unnecessary resolution steps.
- Use `--search-query "<name>"` with `fetch-remote-options` to narrow results when looking up a specific user, space, or list by name.
- **If results are ambiguous** (multiple matches returned), stop and present the options to the user — ask them to confirm which one they mean before proceeding. Never guess or pick the first result.
- Prefer bounding results via action parameters (`page`, `archived`, `limit`) instead of shell post-processing. If you need shell post-processing (`jq`, `grep`, `wc`), wrap the CLI invocation in `sh -c` first:
  ```bash
  sh -c '"$CANVAS_CLI" <subcommand> [flags] --output json' | jq ...
  ```

### DON'T
- **Never** pass `"accountId"` in `--configured-props` — auth is injected automatically by the CLI.
- **Never** pipe `$CANVAS_CLI ...` directly — the SDK Bash tool returns `(eval):1: permission denied:` when a command from an env var appears on the left side of a pipe. Always use the `sh -c` wrapper form shown above.
- **Never** hardcode a `space_id`, `list_id`, or `folder_id` that isn't in `SKILL-CONTEXT.md` — resolve it via `fetch-remote-options` and record it.

## Available Actions

### Tasks
| Key | Description |
|-----|-------------|
| `clickup-context` | Get comprehensive context about the authenticated user, workspaces, lists, and teammates. Use as the first action to populate workspace hierarchy. |
| `clickup-find-tasks` | Primary action for finding tasks. Replaces `get-tasks`, `get-tasks-by-user`, and `get-tasks-in-space`. Supports filtering by space, list, assignee, status, due dates, and text search. Use `summary_only: true` for count queries. |
| `clickup-project-summary` | Get a comprehensive summary of a list including task counts by status, recent comments, and URL. Accepts a list name or numeric ID. |
| `clickup-create-task` | Create a new task in a ClickUp list |
| `clickup-get-task` | Retrieve a single task by its ID |
| `clickup-update-task` | Update an existing task's properties |
| `clickup-delete-task` | Delete a task from the workspace |
| `clickup-get-users` | Get team members/users from the workspace |

### Comments
| Key | Description |
|-----|-------------|
| `clickup-create-task-comment` | Add a comment to a task (optionally notify assignees or tag users) |
| `clickup-get-task-comments` | Retrieve all comments from a task |
| `clickup-create-list-comment` | Add a comment to a list |
| `clickup-get-list-comments` | Retrieve all comments from a list |
| `clickup-update-comment` | Update a comment's text or mark it resolved/unresolved |
| `clickup-delete-comment` | Delete a comment from a task or list (irreversible) |

### Docs
| Key | Description |
|-----|-------------|
| `clickup-search-docs` | Search for docs by name or keyword |
| `clickup-create-doc` | Create a new doc (optionally with an initial page) |
| `clickup-get-doc` | Retrieve a doc's metadata including its page IDs |
| `clickup-get-doc-page` | Retrieve the content of a single page from a doc |
| `clickup-create-doc-page` | Create a new page inside a doc (optionally nested) |
| `clickup-edit-doc-page` | Update a page's name and/or content |

## Common Workflows

All snippets use `<team_id>`, `<list_id>`, etc. as placeholders. Substitute from `SKILL-CONTEXT.md` at execution time.

### Get workspace context (start here if SKILL-CONTEXT.md is empty)

```bash
$CANVAS_CLI direct-execute-action \
  --component-key clickup-context \
  --configured-props '{}' \
  --output json
```

### Fetch tasks for a user

```bash
$CANVAS_CLI direct-execute-action \
  --component-key clickup-find-tasks \
  --configured-props '{"team_id":"<team_id>","assignee":"<name or user_id or email>","page":0}' \
  --output json
```

For a quick count only, use `summary_only: true`:
```bash
$CANVAS_CLI direct-execute-action \
  --component-key clickup-find-tasks \
  --configured-props '{"team_id":"<team_id>","assignee":"<name>","summary_only":true}' \
  --output json
```

### Get all tasks in a list

```bash
$CANVAS_CLI direct-execute-action \
  --component-key clickup-find-tasks \
  --configured-props '{"team_id":"<team_id>","list_id":"<list_id>","page":0}' \
  --output json
```

### Get all tasks in a space

```bash
$CANVAS_CLI direct-execute-action \
  --component-key clickup-find-tasks \
  --configured-props '{"team_id":"<team_id>","space_id":"<space_id>","page":0}' \
  --output json
```

### Get list summary

```bash
$CANVAS_CLI direct-execute-action \
  --component-key clickup-project-summary \
  --configured-props '{"team_id":"<team_id>","list_id":"<list_id_or_name>"}' \
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

When you hit an ID that isn't in `SKILL-CONTEXT.md`, resolve it with one of these and record the result.

### Get workspaces
```bash
$CANVAS_CLI fetch-remote-options \
  --component-key clickup-find-tasks \
  --field-name team_id \
  --output json
```

### Get spaces (requires team_id)
```bash
$CANVAS_CLI fetch-remote-options \
  --component-key clickup-find-tasks \
  --field-name space_id \
  --current-configuration '{"team_id":"<team_id>"}' \
  --output json
```

### Get lists within a space (requires space_id)
```bash
$CANVAS_CLI fetch-remote-options \
  --component-key clickup-find-tasks \
  --field-name list_id \
  --current-configuration '{"team_id":"<team_id>","space_id":"<space_id>"}' \
  --output json
```

### Find a user by name

Use `clickup-get-users` to list all workspace members:
```bash
$CANVAS_CLI direct-execute-action \
  --component-key clickup-get-users \
  --configured-props '{"team_id":"<team_id>"}' \
  --output json
```

> Tip: `clickup-find-tasks` also accepts `assignee` as a name, email, or user ID directly — no prior resolution needed.
