# Mind Map Repo Instructions

This repo stores mind-map content only. Do not add builder code here.

## Canonical References

Before creating or editing any map folder, read the relevant guide in `excali-builder` (likely a sibling directory at `../excali-builder` relative to this repo's root):

- Markdown mind maps: `https://github.com/yeqline/excali-builder/blob/master/docs/mind-map-ai-agent-guide.md`
- CSV diagrams: `https://github.com/yeqline/excali-builder/blob/master/docs/csv-parser-guide.md`
- Builder usage: `https://github.com/yeqline/excali-builder/blob/master/README.md`

Also read `mind-map-manifesto.md` before creating or substantially changing a mind map. Use it for information design, decomposition quality, and avoiding artificial-looking map structures.

## Repo Conventions

- Keep maps under `mind-maps/<domain>/<topic>/`, for example `mind-maps/de/dbt` and `mind-maps/de/snowflake`.
- Keep source files, checked-in config files, `positions.json`, and `output.excalidraw` in this repo.
- Commit both `positions.json` and `output.excalidraw` when layout changes should be preserved across machines.
- Use lowercase kebab-case for new file names, for example `mind-map-manifesto.md`.
- When format conventions conflict with local assumptions, the `excali-builder` docs win.

## Build Workflow

Run builds from the `excali-builder` repo against a map folder in this repo:

```bash
cd /path/to/excali-builder
uv run excali-builder /path/to/mind-map/mind-maps/<domain>/<topic>
```
