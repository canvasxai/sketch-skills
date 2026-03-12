# sketch-skills

Featured skills for [Sketch](https://github.com/canvasxai/sketch), the org-level AI assistant.

## Structure

Each skill lives in `skills/<skill-id>/` and contains:

- `SKILL.md` — defines the skill (name, description, provider type, usage instructions for the agent)
- Optional CLI script (e.g. `canvas-cli.js`) — invoked by the agent to execute skill operations

The `manifest.json` at the root lists all available skills with their metadata.

## Auto-sync

Sketch pulls from this repo on startup, making all listed skills available to the agent automatically.
