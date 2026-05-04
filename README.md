# Mind Maps

This repo stores mind-map source content only. The build logic and technical format rules live in the sibling `excali-builder` repo, so this guide stays intentionally thin.

## Canonical Docs

Use these files in `excali-builder` as the technical source of truth for parser syntax, config files, and build behavior:

- Markdown format authoring: [mind-map-ai-agent-guide.md](https://github.com/yeqline/excali-builder/blob/master/docs/mind-map-ai-agent-guide.md)
- CSV authoring: [csv-parser-guide.md](https://github.com/yeqline/excali-builder/blob/master/docs/csv-parser-guide.md)
- Builder usage: [excali-builder README](https://github.com/yeqline/excali-builder/blob/master/README.md)

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

## Responsibility Boundary

The sibling `excali-builder` repo should stay technical: parser rules, supported syntax, config behavior, and build workflow.

This repo owns the semantic and editorial standards for mind maps: information design, learning quality, decomposition, node type policy, edge usage policy, and guidance about natural branch depth.

## Quick Start

1. Create a map folder under `mind-maps/<domain>/<topic>/`.
2. Choose one format:
   - Markdown: follow the [mind-map AI agent guide](https://github.com/yeqline/excali-builder/blob/master/docs/mind-map-ai-agent-guide.md)
   - CSV: follow the [CSV parser guide](https://github.com/yeqline/excali-builder/blob/master/docs/csv-parser-guide.md)
3. Add the required source and config files for that format.
4. Build from your local `excali-builder` clone:

```bash
cd /path/to/excali-builder
uv run excali-builder /path/to/mind-map/mind-maps/<domain>/<topic>
```

5. Open the generated `output.excalidraw` if you want to adjust layout, then rebuild after content changes.
6. After moving things in Excalidraw, save and rebuild once so the layout is synced back into `positions.json`, then commit both `positions.json` and `output.excalidraw`.

## Recommended Way To Manage Docs

Do not copy the full builder guide into this repo.

- Keep parser and builder standards in `excali-builder`.
- Keep this repo's docs limited to purpose, folder structure, and the exact canonical references.
- Use GitHub links for canonical docs so the references work from any machine.
- Commit `positions.json` and `output.excalidraw` if you want the easiest cross-machine workflow.
- Organize maps under `mind-maps/<domain>/<topic>/`, for example `mind-maps/de/dbt` or `mind-maps/de/snowflake`.
- If parser or builder standards change, update only the canonical guide in `excali-builder`.
- If the doc path changes later, update the path in this README and `AGENTS.md`.
