---
name: sioma-onboard
description: Onboard this repository to Sioma end to end — scan the API, write and publish an OpenAPI spec, and verify the entity graph. Use when the user says to onboard/connect/set up this repo or project with Sioma, wants Sioma to know their API, or asks how to get started with Sioma.
---

You are running Sioma's onboarding skill (v1.5.0). Do the whole loop — do not stop at a file.

Onboard this repository to Sioma: produce a spec of its API, publish it, and verify the
agent can actually see the resulting graph. Work end to end — do not hand the user a
file and stop.

## 0. Where are you?

Find the API first. Look for route files (Next.js `app/**/route.ts` or `pages/api/**`,
express/fastify/hono `app.get(...)`/`router.post(...)`, NestJS controllers, FastAPI/DRF,
Rails routes), plus the types/validators/ORM models behind them. If this repo has no HTTP
API, say so and stop — Sioma builds a graph of an API surface; there is nothing to onboard.

## 1. Enumerate the routes — deterministically if you can

Run Sioma's scanner. It walks the AST and emits a byte-stable openapi.json of the REAL
routes with zero guessing. A fresh repo won't have `sioma` globally installed — that is
the common case, so reach for `npx`, which runs the published CLI with no install:

```sh
sioma scan .                        # if @sioma/cli is already on PATH
npx -y @sioma/cli@latest scan .     # zero-install — the usual first-run path
```

It covers Next.js (App + Pages), express, fastify and hono. Read what it reports.

**In a monorepo, scan each app — not the root.** The scanner detects ONE framework at
the directory you point it at, and a workspace root (a `pnpm-workspace.yaml`, a
`workspaces` field, or an `apps/`+`packages/` layout) is none — `scan .` there finds
ZERO. Point it at each package that actually serves an HTTP API (`apps/server`,
`apps/web`, `services/*`, …), one scan per app, and PUBLISH EACH AS ITS OWN NAMED SOURCE
(use the package path as the name) so they stay distinct — re-publishing the same name
just updates that source, never duplicates it:

```sh
npx -y @sioma/cli@latest scan apps/server --out apps/server/openapi.json
npx -y @sioma/cli@latest scan apps/web    --out apps/web/openapi.json
```

**Then read each emitted openapi.json critically, because a scan is a floor, not a
ceiling.** The scanner only emits what the TypeScript checker can PROVE, and it drops
anything it can't — by design, because a wrong entity is worse than no entity. On a real
codebase it will typically give you every route and method, and few or no response
schemas: it cannot type a handler wrapped in `withAuth(...)`/`withWorkspace(...)`, a
payload that comes back from another module unannotated (`res.json(await getThings())`),
or a Zod schema imported from another file. Those are the common cases, so expect a spec
that is exhaustive on paths and thin on schemas.

**That is your job.** The scan ENUMERATES; you LABEL. Keep every route it found — it
found them from the code, you would have missed some — and fill in what it left empty by
reading the types, validators and ORM models yourself. Never delete a scanned route
because you can't type it; describe it as best the code supports.

If the scanner can't run (no registry access in an air-gapped CI, or the stack isn't
covered — Django, Rails, Go, …), check for a framework-native generator (NestJS Swagger,
drf-spectacular, FastAPI's own /openapi.json) and use that as the enumeration floor
instead. If nothing is available, enumerate the routes by reading the code — exhaustively,
every file, no sampling.

## 2. Write the spec

Use the extract-openapi skill's rules (same plugin) for the details — they are the
contract Sioma's graph builder reads. The parts that decide whether you get a graph at
all, restated because they are where onboarding silently fails:

- Every domain object is a NAMED schema under `components.schemas`. No anonymous inline
  objects for domain nouns.
- **Every 2xx response body `$ref`s a component schema**, or is an array whose `items` is
  a `$ref`. This is what makes an entity. A route with no response schema builds NOTHING —
  that is exactly how a scan of 468 routes produces a graph of zero.
- Declare relationships explicitly: an object `$ref` property is belongs-to, an array of
  `$ref` is has-many. For scalar foreign keys (`userId`) — which a `$ref` cannot express —
  declare them on the list operation with
  `"x-sioma-ai": {"relationships":[{"field":"userId","entity":"user","type":"belongs_to"}]}`,
  where `entity` is the target's singular, lower-case, hyphenated PATH id (`/order-items`
  → `"order-item"`, never `"OrderItem"` — camelCase is not normalized and would silently
  miss).
- Sioma NEVER guesses from a field name. Anything you don't declare does not exist.
- **ASK THE USER for the API's base URL** — one absolute http(s) URL where this API is
  actually served (e.g. `https://api.example.com`). Neither you nor the scanner can know
  a deploy URL from code, and Sioma never infers one: without a declared `servers` entry
  the graph serves context but can run NO flows. Pass it to the scanner
  (`sioma scan . --server https://api.example.com`) or write it as the spec's top-level
  `"servers": [{"url": "..."}]` (exactly one). If the user has no stable URL yet, say so
  honestly and continue — context works either way; flows unlock when a server is declared
  and republished.

Write it to `openapi.json` at the repo root — or, in a monorepo, one per app beside its
package (`apps/server/openapi.json`, …), each published as its own named source.

## Publish it to Sioma — you do this, not the user

Sioma builds the graph from a PUBLISHED spec. A registered spec that was never
published is invisible to the agent: the cell serves nothing and `sioma_list_entities`
returns "No spec is registered". Registering is not publishing — publish, then verify.

### Get credentials — check the environment FIRST

If BOTH `$SIOMA_API_KEY` and `$SIOMA_PUBLISH_KEY` are already exported, skip this
whole section — you are authenticated; go straight to Publish. Also make sure
`SIOMA_APP_URL` points at the workspace origin (default `https://app.sioma.ai`):

```sh
export SIOMA_APP_URL=${SIOMA_APP_URL:-https://app.sioma.ai}
```

If either key is missing, run the OAuth device flow (RFC 8628). A key cannot mint keys,
so the one human act is a single browser click — you do everything else; never ask the
user to mint, copy or paste a key.

1. Request a device code (public endpoint, no auth). Each block below sets
   `SIOMA_APP_URL` on its own line first (a prefix assignment on the curl line
   itself would NOT expand in that same command's URL), so the block stands
   alone — a var exported in a separate command is gone in the next:

```sh
SIOMA_APP_URL=${SIOMA_APP_URL:-https://app.sioma.ai}; curl -fsS -X POST "$SIOMA_APP_URL/oauth/device" \
  -H "content-type: application/json" \
  -d '{"client_name":"<your agent name, e.g. Claude Code>"}'
```

2. Print the response's `verification_uri_complete` (and the `user_code`) and ask
   the user to open it and click **Approve** — in the browser where they are signed
   in to their Sioma workspace. Tell them plainly: only approve a code they just saw
   you print.

3. Poll the token endpoint, waiting `interval` seconds (default 5) between polls:

```sh
SIOMA_APP_URL=${SIOMA_APP_URL:-https://app.sioma.ai}; curl -sS -X POST "$SIOMA_APP_URL/oauth/token" \
  -H "content-type: application/x-www-form-urlencoded" \
  --data-urlencode "grant_type=urn:ietf:params:oauth:grant-type:device_code" \
  --data-urlencode "device_code=<the device_code from step 1>"
```

   `authorization_pending` = keep polling. `slow_down` = you polled too fast — wait
   longer. `expired_token` = the 10-minute window closed; restart from step 1.
   `access_denied` = the user said no; stop and ask.

4. On 200, CHECK `sioma_workspace` in the response and tell the user which workspace
   they connected — e.g. "connected to workspace: Acme". If that is NOT the workspace
   they meant, STOP: someone else approved the code. A device code is only a request;
   whoever clicks Approve binds it to THEIR workspace, so a code that leaked (a shared
   terminal, a pasted log) can be claimed by a stranger. Do not publish until the user
   confirms the workspace is theirs.

   You hold both keys now. A shell `export` does NOT carry into your next command
   (each runs in a fresh shell), so don't rely on `$SIOMA_API_KEY`/`$SIOMA_PUBLISH_KEY`
   resolving in a later step — put each key's actual value into the request that needs
   it, or export and use it in the SAME command. Keep the keys in this session and in
   your agent's own MCP config only; never put them in a file in this repo or anything
   you commit. The agent key (`access_token`) is your MCP credential; the publish key
   is used once, in the next step.

Fallback (no browser reachable): the user can mint the keys manually in their Sioma
workspace — Control room → Keys → New key → scope **publish**, and another with scope
`agent` — and export them as above.

A `publish` key is least-privilege by design: it can upload and publish specs and
nothing else. It is NOT the `agent` key your MCP connection uses, and it is
deliberately excluded from the cell manifest — a control-plane credential must never
become a data-plane one.

### Publish

Prefer the CLI, and publish the `openapi.json` you just wrote with `--spec` — a bare
`sioma publish .` RE-SCANS the repo and would DISCARD every schema and relationship you
hand-added. Use `npx` when it isn't globally installed:

```sh
sioma publish --spec openapi.json                        # if @sioma/cli is on PATH
npx -y @sioma/cli@latest publish --spec openapi.json     # zero-install
```

If you split a monorepo into one spec per app, publish each with its own `--spec` and a
distinct `--name` so each becomes its own source (re-publishing the same name updates it
in place, never duplicates):

```sh
npx -y @sioma/cli@latest publish --spec apps/server/openapi.json --name apps/server
npx -y @sioma/cli@latest publish --spec apps/web/openapi.json    --name apps/web
```

(In CI, where the spec is regenerated fresh and never hand-edited, `sioma diff || sioma
publish` WITHOUT `--spec` re-scans and publishes only when the code changed — that is the
content-hash change-gate.)

Otherwise call the same two endpoints directly (no install needed; needs `jq`, or build
the same body with node/python if `jq` is missing). `spec` is the PARSED JSON document,
not a string. Run this as ONE command so the key resolves (a var set in a separate
command is gone in the next); put the real publish key in the first line. PIPE the body
in with `--data-binary @-` — a big spec on the `-d` argument hits the shell's argument
size limit:

```sh
export SIOMA_PUBLISH_KEY=<the sioma_publish_key value> SIOMA_APP_URL=${SIOMA_APP_URL:-https://app.sioma.ai}
jq -cn --slurpfile spec openapi.json '{name:"openapi.json", kind:"paste", spec:$spec[0]}' \
  | curl -fsS -X POST "$SIOMA_APP_URL/api/anatomy/sources" \
      -H "Authorization: Bearer $SIOMA_PUBLISH_KEY" -H "content-type: application/json" --data-binary @-
curl -fsS -X POST "$SIOMA_APP_URL/api/publish" \
  -H "Authorization: Bearer $SIOMA_PUBLISH_KEY"
```

Both are idempotent and tenant-derived-from-key — there is no tenant id to pass and
no way to publish into someone else's workspace. In a monorepo, repeat the two calls per
app, giving each source POST its own `name` (the app path) and its app's `openapi.json`.

**A 2xx from `/api/publish` is NOT success.** Read the response body:

```json
{ "specHash": "5c19a7ee…", "bytes": 5156, "announced": false }
```

`"announced": false` means the spec was STORED but never signed into a manifest, so no
cell can serve it — the agent will still get "No spec is registered". The call returns
2xx either way. **Require `"announced": true` before you report success.** If it is
false, the workspace's control plane has no manifest signing key configured; that is an
operator fix, not something to retry — say so and stop.

### Verify — do NOT skip this

Publishing "succeeding" is not proof. The spec has to reach the cell as a SIGNED
manifest before an agent sees it. Confirm with the graph itself:

- If you are connected to Sioma over MCP, call **`sioma_list_entities`** and report the
  count and the entity names.
- A NON-EMPTY list is the pass signal — not an exact number. The agent's list includes
  action endpoints that aren't standalone entities, which the dashboard filters out, so
  the agent's count is normally HIGHER than the dashboard's; a larger count is expected,
  not a mismatch. The only failures are ZERO entities or "No spec is registered".

Then say plainly what you got:

- **Entities listed** — the loop is closed; the agent is serving from the real graph.
- **"No spec is registered"** — the publish did NOT land. Do not paper over this. Check
  that `/api/publish` returned 2xx and that the workspace shows an announced spec hash.
- **A publish 2xx but ZERO entities** — the spec published fine and built nothing. That
  is a SPEC problem, not a publish problem: Sioma's anatomy is explicit-only, so a spec
  whose 2xx responses don't `$ref` named component schemas materializes no entities.
  Re-read the graph requirements above and fix the spec, then re-publish. Report the
  number honestly — a graph of 0 entities is a failure even though every call returned 200.

### Rate limits / caps

Spec mutations are capped at 20/min per tenant and 50 sources per workspace. If you get
a 429, wait — do not retry in a loop.

## Connect the agent (if it isn't already)

Publishing gets the graph built; connecting is what lets your agent USE it. If Sioma's
MCP tools aren't available to you, you need the `agent` key (the `access_token` from the
device flow, a different key from the `publish` one) registered against the cell's
`/mcp` endpoint. In Claude Code, installing the Sioma plugin does this; without it, run
`claude mcp add --transport http sioma "<cellUrl>/mcp" --header "Authorization: Bearer <the
access_token value>"`. Other agents write their USER-level MCP config (Cursor
`~/.cursor/mcp.json`, Windsurf `~/.codeium/windsurf/mcp_config.json`, VS Code user
settings) with the same URL and header — never a project file inside this repo. MCP
config loads at startup, so BEFORE the restart, give the user the message to send you
afterward ("Call sioma_list_entities and report the count"); in Claude Code have them
restart with `claude --continue`. After the restart, call `sioma_list_entities` to confirm.

## If rails is on: ride, don't hand-call

If the tool list includes `sioma_board`, this workspace has rail mode on: when a
`sioma_focus` response carries `rails`, prefer BOARDING one over making the API calls
yourself — `sioma_board` with the rail's `entityId` (+ the `recordId` from that focus, so
the completed ride records its outcome), then `sioma_provide` for any pending slot. Sioma
makes the calls; you get the step results. Read flows are compiled from the
declared spec and run on their own — a step that needs an API key the workspace hasn't
added halts honestly; report that rather than working around it.

A WRITE flow (`sioma_board` with a non-GET `method`) never runs on its own: the session
pauses in `awaiting-approval` until the workspace owner approves that exact flow. Show
the user the approval `url` from the response and poll `sioma_inspect` — the ride resumes
by itself once approved, or halts if denied. Never retry `sioma_board` to get past the
pause, and never make the write calls yourself instead.

## Report

State, plainly and with numbers:

1. routes found in the code vs operations described in the spec (if these differ, say why);
2. schemas written, and how many 2xx responses `$ref` one;
3. entities and relationships `sioma_list_entities` actually returns after publish;
4. anything you could not type or declare, and why.

If (3) is zero, the onboarding FAILED even if every command returned 200 — say so.
