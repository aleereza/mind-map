# Mind Maps

This repo contains Markdown-based mind maps that are converted into Excalidraw diagrams with `excali-builder`. Each map keeps its learning content in source Markdown, its visual styling in checked-in config files, and its reusable layout in `positions.json`.

## Repo Layout

Mind maps live under `mind-maps/`, grouped by domain and then topic:

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

Each map folder includes the Markdown source, checked-in config files, optional `positions.json`, and generated `output.excalidraw`. Keeping `positions.json` and `output.excalidraw` in the repo gives the easiest cross-machine continuity: `positions.json` stores durable layout state, and `output.excalidraw` is useful for direct Excalidraw use and as a backup source for sync.

## Creation Workflow

The main workflow pairs an AI agent for Markdown source edits with the live `excali-builder serve` viewer for review and layout:

1. A new map starts as a folder under `mind-maps/<domain>/<topic>/`.
2. An AI agent can create the initial Markdown source and config files for the subject.

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

3. The map folder contains the required Markdown source and config files.
4. The local auto-refreshing viewer runs from the builder:

```bash
cd /path/to/excali-builder
uv run excali-builder serve /path/to/mind-map/mind-maps/<domain>/<topic>
```

5. The local viewer URL printed by `serve` opens the generated Excalidraw map in the browser.
6. Node repositioning and resizing happen directly in the viewer, and layout saves automatically to `positions.json`.
7. While `serve` keeps running, AI-agent edits to Markdown or config files trigger automatic rebuilds.
8. The viewer refreshes without a manual reload or separate build command.
9. Source files, config files, `positions.json`, and `output.excalidraw` are committed when the map is in a useful state.

In this workflow, source files are authoritative for content and relationships, while `positions.json` is authoritative for durable layout. The local viewer is for spatial review and node layout, not for editing the canonical text content.

## One-Shot Build

The original build command still works for scripts or export-only work:

```bash
cd /path/to/excali-builder
uv run excali-builder /path/to/mind-map/mind-maps/<domain>/<topic>
```

A one-shot build syncs layout from any existing `output.excalidraw`, regenerates the diagram, and exits. The `serve` workflow is better suited to day-to-day map creation and review.
