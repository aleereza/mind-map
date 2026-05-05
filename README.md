# Mind Maps

This repo stores mind-map source content only. This guide stays focused on folder layout and the day-to-day creation workflow.

## Recommended Repo Layout

Keep all content under `mind-maps/`. Organize by domain, then topic:

```text
mind-maps/
  de/
    dbt/
      config.json
      node_config.json
      edge_config.json
      dbt.md
      positions.json
      output.excalidraw
    snowflake/
      config.json
      node_config.json
      edge_config.json
      snowflake.md
```

Keep source files, config files, `positions.json`, and `output.excalidraw` in this repo if you want seamless cross-machine continuity. `positions.json` is the durable layout state used by the builder, and `output.excalidraw` is useful as the file you open directly in Excalidraw and as a backup source for sync.

## Quick Start

The main workflow is to pair an AI agent for source edits with the live `excali-builder serve` viewer for review and layout:

1. Create a map folder under `mind-maps/<domain>/<topic>/`.
2. Ask an AI agent to create the initial mind map source files for the subject.

Sample prompt:

```text
This repo stores learning-oriented mind maps as Markdown source files. The Markdown files use a special heading-and-anchor format that `excali-builder` converts into Excalidraw diagrams.

Before creating or editing files, read:
- `README.md`
- `AGENTS.md`
- `mind-map-manifesto.md`

Follow `AGENTS.md` for any required technical references.

Create a new mind map for `<subject>` under:

    mind-maps/<domain>/<topic>/

Use Markdown only. Create the required files:
- one kebab-case `.md` source file with globally unique `{#node-id}` anchors
- `config.json`
- `node_config.json`
- `edge_config.json`

Optimize the map for teaching and cognitive clarity, not exhaustive coverage. Keep the first layer small, make sibling nodes parallel, let branch depth follow meaning, and use cross-links sparingly.
```

3. Add or verify the required Markdown source and config files.
4. Start the local auto-refreshing viewer:

```bash
cd /path/to/excali-builder
uv run excali-builder serve /path/to/mind-map/mind-maps/<domain>/<topic>
```

5. Open the local viewer URL printed by `serve`.
6. Review the generated Excalidraw map in the browser.
7. Reposition or resize nodes directly in the viewer; layout saves automatically to `positions.json`.
8. Ask the AI agent to edit the Markdown or config files while `serve` keeps running.
9. The server rebuilds automatically and the viewer refreshes without a manual reload or separate build command.
10. Commit source files, config files, `positions.json`, and `output.excalidraw` when the map is in a useful state.

In this workflow, source files are authoritative for content and relationships, while `positions.json` is authoritative for durable layout. The local viewer is for spatial review and node layout, not for editing the canonical text content.

## One-Shot Build

The original build command still works for scripts or export-only work:

```bash
cd /path/to/excali-builder
uv run excali-builder /path/to/mind-map/mind-maps/<domain>/<topic>
```

A one-shot build syncs layout from any existing `output.excalidraw`, regenerates the diagram, and exits. Prefer `serve` for day-to-day map creation and review.

## Recommended Way To Manage Docs

Do not copy the full builder guide into this repo.

- Keep Markdown parser and builder standards out of this README.
- Keep this repo's docs limited to purpose, folder structure, and practical workflow.
- Commit `positions.json` and `output.excalidraw` if you want the easiest cross-machine workflow.
- Organize maps under `mind-maps/<domain>/<topic>/`, for example `mind-maps/de/dbt` or `mind-maps/de/snowflake`.
- If parser or builder standards change, update the canonical technical docs and `AGENTS.md`, not this README.
