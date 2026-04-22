# ClickUp — Org Context

This file holds **org-specific** ClickUp data used by the `clickup` skill: workspace `team_id`, users, lists, and spaces that you touch often enough to be worth memorizing.

It's seeded once when the skill is first installed and is **not overwritten** by subsequent skill syncs — your edits are preserved. Treat it as a live notebook: whenever the agent resolves a new stable ID, it should append the entry here so future sessions skip the lookup.

## Rules

- **Only stable entities.** Workspaces, users, spaces, folders, lists. Never record task IDs or doc IDs — they churn too fast and will mislead future sessions.
- **Keep names human-readable.** The `Name` column is what the agent matches on when the user says "move this to the Roadmap list" or "assign to Priya".
- **One row per entity.** If a name collides (two users named Alex), disambiguate in the Name column (`Alex R.`, `Alex K.`).
- **Remove stale rows** when a user leaves or a list is archived — a wrong ID here is worse than no ID.

## Workspaces
| Name | team_id |
|------|---------|
| _(add workspaces as discovered)_ | |

## Users
| Name | user_id |
|------|---------|
| _(add users as discovered)_ | |

## Lists
| Name | list_id |
|------|---------|
| _(add lists as discovered)_ | |

## Spaces
| Name | space_id |
|------|----------|
| _(add spaces as discovered)_ | |

## Folders
Folders sit between spaces and lists. Record the parent `space_id` so the agent can re-derive the hierarchy without another lookup.

| Name | folder_id | space_id |
|------|-----------|----------|
| _(add folders as discovered)_ | | |
