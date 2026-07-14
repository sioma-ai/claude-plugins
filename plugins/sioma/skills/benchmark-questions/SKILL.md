---
name: benchmark-questions
description: Write ground-truthed benchmark questions over the tenant's Sioma entity graph, in the exact JSON shape the Sioma benchmark upload accepts. Use when the user wants benchmark questions, a test suite for Sioma, or to measure Sioma's context savings.
---

You are running Sioma's benchmark-questions skill (prompt v1.0.0) — write the suite, then hand it to the user for upload.

Write benchmark questions for my Sioma workspace. Sioma has built a typed entity graph from my OpenAPI spec; the benchmark measures whether, for each question, Sioma serves the right minimal sub-context. I will paste your output into the Sioma benchmark's question upload, which validates every row against my real entity graph.

First, look at my entity graph (if the Sioma MCP server is connected, call sioma_list_entities and sioma_explore_entity; otherwise ask me to paste the entity list from my Sioma workspace's Entities page). You need the REAL entity ids — a question whose expectedEntityId is not an entity in the graph is rejected on upload.

Then write questions that sound like real tasks in this domain, not like schema lookups:

- Simple lookups: a task that lands on one entity ("find the customer who…").
- Relation chains: tasks that must walk a declared relationship or two ("the items on the latest invoice of the customer who…") — the ground truth is the entity the answer LIVES on, at the end of the chain.
- Business scenarios: a paragraph-length situation a real user would bring, whose answer still grounds to one entity.
- No-answer questions (2–3 of them): plausible-sounding asks the API genuinely cannot answer — the ground truth is "" and Sioma should red-flag them rather than serve a wrong map.

Output contract — a single JSON array, nothing else:

[
  { "intent": "<the question>", "expectedEntityId": "<an entity id from MY graph, or \"\" for no-answer>" }
]

Example (with a graph whose entities include Customer and InvoiceItem):

[
  {
    "intent": "Which customer placed order #4821, and what company are they with?",
    "expectedEntityId": "Customer"
  },
  {
    "intent": "Show the line items on the most recent invoice for Acme Corp",
    "expectedEntityId": "InvoiceItem"
  },
  {
    "intent": "What is the current weather in Berlin?",
    "expectedEntityId": ""
  }
]

Rules: each intent under 500 characters; at most 500 rows (a good suite is 12–48); expectedEntityId must be byte-for-byte an id from my graph (casing is forgiven, nothing else) or "" for no-answer; no duplicate intents; do not invent entities. When you are done, hand me the JSON array and remind me to upload it via the benchmark panel's question manager.

When the JSON array is ready, point the user at the benchmark panel's question manager in their Sioma workspace to upload it.
