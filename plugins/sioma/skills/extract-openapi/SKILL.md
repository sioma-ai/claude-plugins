---
name: extract-openapi
description: Generate or repair an OpenAPI spec for this repository so Sioma can build its entity graph. Use when the user wants to extract/produce/fix an openapi.json for Sioma, onboard this repo to Sioma, or asks how to get their API into Sioma.
---

You are running Sioma's extract-openapi skill (prompt v1.0.0) — produce the spec, then walk the user to the upload.

Produce an OpenAPI 3.x spec for this repository's API, as a single JSON file named openapi.json. The spec will be ingested by Sioma, a context engine that builds a typed entity graph from it — the graph is only as good as the spec's declared schema relationships, so follow the hard requirements below exactly.

Work in this order:

1. Look for an existing OpenAPI/Swagger document first (openapi.json, openapi.yaml, swagger.json, files under docs/, a spec-generation script, or a framework that can emit one). If you find one, validate it against the hard requirements below, repair what falls short, and emit it — do not rebuild from scratch.

2. If none exists, detect the stack (Next.js, Express, Fastify, NestJS, Hono, Koa, Django, Rails, Spring, Go net/http, …) and enumerate the real routes from the code — router registrations, controller decorators, file-based route dirs. Do not invent endpoints that are not in the code.

3. Derive schemas from the real sources of truth, in preference order: TypeScript types, zod/valibot/joi validators, Prisma/Drizzle/ORM models, serializer classes, then database migrations. Name each schema after its domain noun (User, Order, Invoice — singular PascalCase).

HARD REQUIREMENTS — Sioma's anatomy is explicit-only; it never guesses from field names. A spec that ignores these builds ZERO entities:

- Every domain object is a named schema under components.schemas. No anonymous inline object schemas for domain nouns.
- Every 2xx response body is a $ref to a components schema, or an array whose items is a $ref. A list endpoint returns { "type": "array", "items": { "$ref": "#/components/schemas/X" } }.
- Relationships are expressed as $ref fields ON the schemas: a field holding an object $ref declares belongs-to; a field holding an array of $ref declares has-many. Never inline a copy of a related object — always $ref the named schema.
- Keep id fields and human-display fields (name/title/email/…) on every schema.
- Paths follow the resource hierarchy the code actually implements (e.g. /users/{userId}/orders), because the path tree is also part of the graph.

4. Self-check before emitting, and fix anything that fails:
- Does every list endpoint return an array of $ref?
- Does every domain schema live in components.schemas and get $ref'd (never inlined) everywhere it appears?
- Is every relationship you know exists in the domain expressed as a $ref field on a schema?
- Did you invent any endpoint or field that is not in the code? Remove it.

5. Output: write the complete spec to openapi.json at the repo root — one JSON document, nothing else in the file. Then tell me the entity count you expect (the number of component schemas) and list the relationships you declared, so I can verify them against the graph Sioma builds after I upload it.

## After the spec is written

Tell the user to upload openapi.json in their Sioma workspace (Get started → Add a spec). If this agent is not connected to Sioma yet, the manual MCP connect fallback is:

    claude mcp add --transport http sioma "${SIOMA_CELL_URL:-https://aws-us1.cells.sioma.ai}/mcp" --header "Authorization: Bearer $SIOMA_API_KEY"
