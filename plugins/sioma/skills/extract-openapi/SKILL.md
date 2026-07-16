---
name: extract-openapi
description: Generate or repair an OpenAPI spec for this repository so Sioma can build its entity graph. Use when the user wants to extract/produce/fix an openapi.json for Sioma, or asks how to get their API into Sioma.
---

You are running Sioma's extract-openapi skill (prompt v3.0.0) — produce the spec, then publish it yourself.

Produce an OpenAPI 3.x spec for this repository's API, as a single JSON file named openapi.json. Sioma ingests it to build a typed entity graph AND to serve your agent the minimal context per task — so the spec must be BOTH complete (every endpoint, input, and field the API really has) AND graph-shaped (named schemas + declared relationships). A thin spec builds a thin, useless graph; an incomplete spec leaves your agent blind to half the API. Do both.

PREFER A DETERMINISTIC SCAN — this prompt is the FALLBACK. If a static analyzer is available, RUN IT FIRST instead of writing the spec by hand: the Sioma CLI (`sioma scan`, which emits a byte-stable openapi.json from your real routes + TS types with zero guessing), or your framework's own spec generator (NestJS/FastAPI/drf-spectacular/…). A deterministic scan is repeatable and CI-friendly — a re-run changes only when the CODE changed, which is exactly what Sioma's publish loop wants. With a scanned spec, use the steps below ONLY to fill the semantic x-sioma-ai fields and repair anything the scanner left unknown. Follow the full instructions below when no scanner covers this stack and you must generate the spec by hand.

Work in this order.

1. FIND AN EXISTING SPEC FIRST. Look for openapi.json / openapi.yaml / swagger.json, files under docs/, a spec-generation script, or a framework that can emit one. If you find one, do NOT rebuild from scratch — validate it against the requirements below, repair what falls short (usually: missing relationships, stub schemas, absent request bodies), and emit it.

2. ENUMERATE EVERY ROUTE — be exhaustive, this is where specs go thin. Detect the stack (Next.js, Express, Fastify, NestJS, Hono, Koa, Django, Rails, Laravel, Spring, Go net/http, …) and find ALL routes, not just the obvious ones:
   - every router file, and every sub-router MOUNTED into another (follow app.use('/x', router) / route groups / include() — apply the mount prefix);
   - versioned prefixes (/v1, /api), controller decorators, file-based route dirs (app/**/route.ts, pages/api/**), and dynamically/loop-registered routes;
   - EVERY HTTP method on each path (GET, POST, PUT, PATCH, DELETE) — a resource with a list, a create, and a delete is three operations, not one.
   Reconcile across all router files; if you find N route registrations in the code, the spec should describe ~N operations. Never invent an endpoint that isn't in the code.

3. DERIVE FULL SCHEMAS from the real sources of truth, in order: TypeScript types/interfaces, zod/valibot/joi/yup validators, Prisma/Drizzle/TypeORM/SQLAlchemy/ActiveRecord models, serializer/DTO classes, then database migrations. Name each after its domain noun (User, Order, Invoice — singular PascalCase). Capture the WHOLE object, not a stub: every property, with its type, format (date-time, email, uuid…), enum values, nullability, and which are required. id and human-display fields (name/title/email) must be present, but they are the floor, not the ceiling.

4. CAPTURE INPUTS AND METADATA:
   - requestBody for every write endpoint (POST/PUT/PATCH), as a $ref to a named schema where it's a domain object;
   - path, query, and header parameters (pagination, filters, ids) with types;
   - security: define components.securitySchemes (bearer/apiKey/oauth2 as the code uses) and reference it on protected operations;
   - representative error responses (400/401/404/409/5xx) so the API's failure surface is described.

GRAPH REQUIREMENTS — Sioma's anatomy is EXPLICIT-ONLY; it never guesses from field names. Get these exactly right or entities/edges silently vanish:

- Every domain object is a NAMED schema under components.schemas. No anonymous inline object schemas for domain nouns.
- Every 2xx response body is a $ref to a component schema, or an array whose items is a $ref (a list endpoint returns { "type": "array", "items": { "$ref": "#/components/schemas/X" } }). This is what makes an entity. If the API wraps results in an envelope ({ "data": [...], "total": N }), describe the envelope FAITHFULLY — a $ref (or array of $ref) ONE level inside the envelope still materializes the entity; never flatten a real envelope into a bare array, and don't bury the $ref deeper than one level.
- Paths follow the resource hierarchy the code implements (/users/{userId}/orders) — the path tree is part of the graph. Prefer resource-noun paths so each becomes an entity.
- DECLARE RELATIONSHIPS — two ways, use whichever fits:
  · Object $ref field ON a schema: a property whose value is an object $ref declares belongs-to; an array of $ref declares has-many. Use this when the API actually nests the related object.
  · Scalar foreign keys (userId, tenant_id, order_id: a bare string/number that IDENTIFIES another entity but isn't a $ref) are the COMMON real case and CANNOT be expressed as a $ref. Declare them INSIDE the list operation object (next to "responses", NOT at the path-item level) with the vendor extension:
      "x-sioma-ai": { "relationships": [ { "field": "userId", "entity": "user", "type": "belongs_to" }, { "field": "comments", "entity": "comment", "type": "has_many" } ] }
    where "entity" is the TARGET's lower-case, hyphenated, SINGULAR id — the singularized PATH name, not the schema name: /users → "user", and for a multi-word resource /order-items → "order-item" (NOT "OrderItem" — camelCase is not normalized, so it would silently miss). This is declared signal, not a guess. Make component names singularize to the same id as their resource path, or the target won't resolve; when unsure, use x-sioma-ai.relationships with the explicit target id.
- NON-CRUD / ACTION / RPC endpoints (POST /reports/generate, /auth/login, webhooks, search, bulk ops) don't map to a REST collection, so they'd otherwise build no entity. Keep them (do NOT drop them — the agent needs them), give them multi-segment action paths, and if they read/write a domain object declare it so it still materializes: "x-sioma-ai": { "dataEntities": ["Report"] } on the operation.

SELF-CHECK before emitting — run BOTH passes and fix anything that fails:
A. Coverage — did I capture EVERY route and method I found in the code (list the count found vs described)? Every write endpoint's requestBody? Path/query params? Security schemes? Representative errors? Full property sets (enums/formats/required), not id+name stubs? Did I invent anything not in the code (remove it)?
B. Graph — does every list endpoint return an array of $ref? Does every domain schema live in components.schemas and get $ref'd (never inlined)? Is every relationship I know exists declared (object $ref OR x-sioma-ai.relationships with a resolvable target id)? Do action endpoints carry x-sioma-ai.dataEntities?

OUTPUT: write the complete spec to openapi.json at the repo root — one JSON document, nothing else in the file. Then report, so I can verify against the graph Sioma builds: (a) how many operations/paths and schemas the spec has vs how many routes you found in the code, and (b) the entities you expect and the relationships you declared.

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

1. Request a device code (public endpoint, no auth):

```sh
curl -fsS -X POST "$SIOMA_APP_URL/oauth/device" \
  -H "content-type: application/json" \
  -d '{"client_name":"<your agent name, e.g. Claude Code>"}'
```

2. Print the response's `verification_uri_complete` (and the `user_code`) and ask
   the user to open it and click **Approve** — in the browser where they are signed
   in to their Sioma workspace. Tell them plainly: only approve a code they just saw
   you print.

3. Poll the token endpoint, waiting `interval` seconds (default 5) between polls:

```sh
curl -sS -X POST "$SIOMA_APP_URL/oauth/token" \
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

   Then export both keys and continue. Never write either key into a file — the
   environment is their only home:

```sh
export SIOMA_API_KEY=<the access_token value>           # agent key — your MCP credential
export SIOMA_PUBLISH_KEY=<the sioma_publish_key value>  # publish key — used below
```

Fallback (no browser reachable): the user can mint the keys manually in their Sioma
workspace — Control room → Keys → New key → scope **publish**, and another with scope
`agent` — and export them as above.

A `publish` key is least-privilege by design: it can upload and publish specs and
nothing else. It is NOT the `agent` key your MCP connection uses, and it is
deliberately excluded from the cell manifest — a control-plane credential must never
become a data-plane one.

### Publish

Prefer the CLI when `sioma` is on PATH — it scans, uploads and publishes in one step,
and caches the content hash so a re-run when the code hasn't changed is a no-op (that
is what makes it a CI gate: `sioma diff || sioma publish`):

```sh
sioma publish .
```

Otherwise call the same two endpoints directly — this always works, no install needed.
Note `spec` is the PARSED JSON document, not a string:

```sh
curl -fsS -X POST "$SIOMA_APP_URL/api/anatomy/sources" \
  -H "Authorization: Bearer $SIOMA_PUBLISH_KEY" \
  -H "content-type: application/json" \
  -d "$(jq -c --slurpfile spec openapi.json -n '{name:"openapi.json", kind:"paste", spec:$spec[0]}')"

curl -fsS -X POST "$SIOMA_APP_URL/api/publish" \
  -H "Authorization: Bearer $SIOMA_PUBLISH_KEY"
```

Both are idempotent and tenant-derived-from-key — there is no tenant id to pass and
no way to publish into someone else's workspace.

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
- The count must match what the user's dashboard shows.

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

## If you are not connected to Sioma over MCP

You can still publish (that only needs `$SIOMA_PUBLISH_KEY`), but you won't be able to run
the `sioma_list_entities` verification. Wire up the connection with an `agent`-scoped key
and re-verify — in Claude Code:

    claude mcp add --transport http sioma "${SIOMA_CELL_URL:-https://aws-us1.cells.sioma.ai}/mcp" --header "Authorization: Bearer $SIOMA_API_KEY"
