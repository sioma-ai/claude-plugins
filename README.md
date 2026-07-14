# Sioma — Claude Code plugin

> **Generated — do not edit.** This tree is emitted from the Sioma monorepo
> (`apps/platform/src/plugin-bundle.ts`); edits made here are overwritten on
> the next emit.

This repository is a Claude Code plugin marketplace for [Sioma](https://sioma.ai) —
the context engine that serves your coding agent the minimal sub-context each
task needs, built from your own API's entity graph.

## Install

In Claude Code:

```
/plugin marketplace add sioma-ai/claude-plugins
/plugin install sioma@sioma
```

## Environment

```sh
export SIOMA_API_KEY=...   # required — mint it in the Sioma dashboard (scope: agent)
export SIOMA_CELL_URL=...  # optional — defaults to the shared managed cell;
                           # dedicated-cell tenants set their own cell URL
```

## What installing does

- Registers the Sioma MCP server (your cell's `/mcp` endpoint), authenticated
  with your key via env expansion — no secret ever lands in a file.
- Adds two skills: `extract-openapi` (generate or repair an `openapi.json` so
  Sioma can build your entity graph) and `benchmark-questions` (write
  ground-truthed benchmark questions over your graph).

If `SIOMA_API_KEY` is not exported, the MCP server still loads but every call
returns 401 until you export the variable and restart Claude Code.
