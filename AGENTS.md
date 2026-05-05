# Mind Map Repo Instructions

This repo stores Markdown mind-map content only. Do not add builder code here.

## Canonical References

Before creating or editing any map folder, read the relevant technical guide in `excali-builder` (likely a sibling directory at `../excali-builder` relative to this repo's root):

- Markdown mind maps: `https://github.com/yeqline/excali-builder/blob/master/docs/mind-map-ai-agent-guide.md`
- Builder usage: `https://github.com/yeqline/excali-builder/blob/master/README.md`

Also read `mind-map-manifesto.md` before creating or substantially changing a mind map. This repo, not `excali-builder`, owns semantic and editorial guidance: information design, decomposition quality, node type policy, edge usage policy, and avoiding outline-template behavior. Do not force branches to have the same depth, and do not force them to have different depths; let each branch stop where its real decomposition stops.

## Repo Conventions

- Keep maps under `mind-maps/<domain>/<topic>/`, for example `mind-maps/de/dbt` and `mind-maps/de/snowflake`.
- Keep source files, checked-in config files, `positions.json`, and `output.excalidraw` in this repo.
- Commit both `positions.json` and `output.excalidraw` when layout changes should be preserved across machines.
- Use lowercase kebab-case for new file names, for example `mind-map-manifesto.md`.
- When format conventions conflict with local assumptions, the `excali-builder` docs win.

## Serve Workflow

Use the live local viewer for day-to-day map creation, review, and layout:

```bash
cd /path/to/excali-builder
uv run excali-builder serve /path/to/mind-map/mind-maps/<domain>/<topic>
```

The `serve` workflow should be preferred when the user wants to see edits reflected in the diagram. It rebuilds when source or config files change, refreshes the local viewer automatically, and saves node repositioning/resizing from the viewer back into `positions.json`.

## One-Shot Build

Use a one-shot build for scripts, export-only work, or when no live viewer is needed:

```bash
cd /path/to/excali-builder
uv run excali-builder /path/to/mind-map/mind-maps/<domain>/<topic>
```
