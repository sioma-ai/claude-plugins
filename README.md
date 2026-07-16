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

Then just tell your agent:

```
onboard this repo to Sioma
```

## Environment

```sh
export SIOMA_API_KEY=...      # to USE the graph (scope: agent) — the device flow exports it for you
export SIOMA_PUBLISH_KEY=...  # to PUBLISH a spec (scope: publish) — same flow, same click
export SIOMA_APP_URL=...      # your workspace origin (default https://app.sioma.ai)
export SIOMA_CELL_URL=...     # optional — defaults to the shared managed cell;
                              # dedicated-cell tenants set their own cell URL
```

**You don't mint these by hand.** If the keys aren't already exported, the skills run
the OAuth device flow (RFC 8628): the agent prints a short code and a link, you click
**Approve** once in your signed-in browser, and both keys are delivered straight to
the agent — nothing to copy. Manual minting in the dashboard (Keys → New key) remains
as a fallback.

Two keys, deliberately. `agent` is the data-plane key your MCP connection uses;
`publish` is a least-privilege control-plane key that can upload and publish specs
and nothing else. Publish keys are excluded from the cell manifest, so one can
never become the other. Approval is the single human step — a key cannot mint keys.

## What installing does

- Registers the Sioma MCP server (your cell's `/mcp` endpoint), authenticated
  with your key via env expansion — no secret ever lands in a file.
- Adds three skills:
  - `sioma-onboard` — the whole loop: scan the API, write the spec, publish it,
    and verify the graph. This is the one you want.
  - `extract-openapi` — generate or repair an `openapi.json` (and publish it).
  - `benchmark-questions` — write ground-truthed benchmark questions over your graph.

If `SIOMA_API_KEY` is not exported, the MCP server still loads but every call
returns 401 until you export the variable and restart Claude Code.

## Honest limits

- `sioma scan` (the deterministic AST scanner) enumerates routes well but types
  few of them on a real codebase: it can't see through higher-order route wrappers
  (`withAuth(handler)`), unannotated cross-module payloads, or cross-file Zod. So
  the skill uses the scan to ENUMERATE and the model to LABEL. The CLI is not yet
  on npm.
- A spec whose 2xx responses don't `$ref` named schemas builds ZERO entities, even
  though every call returns 200. The skills are written to report that honestly
  rather than declare success.

## Publish from CI/CD (keep the graph in sync)

Sioma can keep your entity graph current automatically: on every change, regenerate
the spec and publish it. Mint a **`publish`-scoped key** once in the dashboard
(Keys → scope `publish` — least privilege: it can upload + publish a spec and
nothing else — it can't mint keys, read data, or act at your cell) and store it as
a CI secret `SIOMA_PUBLISH_KEY`. Then a pipeline step:

```sh
# 1. (re)generate openapi.json — the extract-openapi skill, or your own build step
# 2. upload it + publish; both are idempotent (unchanged spec → no-op)
curl -fsS -X POST "$SIOMA_APP_URL/api/anatomy/sources" \
  -H "Authorization: Bearer $SIOMA_PUBLISH_KEY" -H 'content-type: application/json' \
  -d "{\"name\":\"ci\",\"kind\":\"paste\",\"spec\":$(cat openapi.json)}"
curl -fsS -X POST "$SIOMA_APP_URL/api/publish" -H "Authorization: Bearer $SIOMA_PUBLISH_KEY"
```

(`SIOMA_APP_URL` = your Sioma control plane, e.g. https://app.sioma.ai.) The tenant
is derived from the key, never sent in the request — so the key only ever touches
its own workspace.
