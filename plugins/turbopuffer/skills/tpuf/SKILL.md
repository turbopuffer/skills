---
description: >
  Use when the user wants to work with turbopuffer — a serverless vector and
  full-text search database. Covers setup, querying (vector ANN, BM25, hybrid),
  writing data, exploring namespaces, and diagnosing issues. Trigger on any
  mention of turbopuffer, vector search, namespace, embeddings, BM25, or
  semantic search in the context of turbopuffer.
allowed-tools: mcp__turbopuffer__search_docs
---

# Turbopuffer

## Preflight

Before any operation, verify:
1. **MCP tools available** — check that `execute` and `search_docs` tools exist. If not:
   > Set your API key and restart Claude Code:
   > ```
   > echo 'export TURBOPUFFER_API_KEY=tpuf_...' >> ~/.zshrc && source ~/.zshrc
   > ```
   > Get your key at https://turbopuffer.com/dashboard
2. **API key valid** — run a quick test:
   ```typescript
   async function run(client) {
     for await (const ns of client.namespaces()) { console.log(ns.id); break; }
     console.log("Connected.");
   }
   ```
3. **Namespace exists** (if user provides one) — verify before querying/writing.

## Routing

Match the user's request to a reference and load it before proceeding:

| User wants to... | Load reference |
|---|---|
| Set up turbopuffer, install SDK, configure API key | `references/setup.md` |
| List namespaces, inspect schema, see what data exists | `references/explore.md` |
| Search, query, find similar, retrieve data | `references/query.md` |
| Add, update, delete data, configure schema | `references/write.md` |
| Fix 429 errors, rate limiting, concurrency issues | `references/troubleshoot-429.md` |
| Load large datasets, bulk ingest, monitor progress | `references/bulk-ingestion.md` |
| Design namespaces, plan sharding, sizing | `references/namespace-design.md` |
| Diagnose slow/empty/wrong queries, optimize schema, fix latency | `references/doctor.md` |
| Stuck indexing, unindexed backlog, indexing throttling | `references/doctor-indexing.md` |

For multi-step requests, load references in sequence as needed.

## Composition patterns

- **First integration**: setup → write (seed example data) → query (test it works)
- **Debug 429s**: troubleshoot-429 (identify cause) → doctor (check cache/latency) → namespace-design (consider sharding)
- **Debug slow/broken queries**: doctor (diagnose) → query (test optimized version)
- **Bulk data load**: namespace-design (plan sharding) → bulk-ingestion (load data) → doctor-indexing (monitor progress)
- **Schema optimization**: explore (inspect) → doctor (audit schema) → write (update schema)
- **New search feature**: explore (discover attributes) → query (build search) → write (add FTS if needed)

## Execution rules

1. **Always check schema before querying** — never guess attribute names.
2. **Confirm destructive actions** — for `delete_by_filter` or `ns.deleteAll()`, confirm intent and show what will be affected before executing.
3. **After writes, verify** — query back a sample to confirm the write succeeded.
4. **Batch writes** — never send one document at a time. Always batch.
5. **Use `limit`, not `top_k`** — `top_k` is deprecated.

## User-only commands (NEVER execute without explicit user confirmation)

- `ns.deleteAll()` — permanently deletes an entire namespace and all its data
- `delete_by_filter` on broad filters — could delete thousands of rows unexpectedly

## Response format

Always structure responses as:
1. **What was done** — action and scope (e.g., "Queried namespace `prod-articles` with BM25 search")
2. **The result** — data, metrics, key output
3. **What to do next** — suggested follow-up or confirmation that the task is complete

## Gotchas

- `distance_metric` is set on first write and **cannot be changed**. If wrong, create a new namespace.
- Namespaces are implicitly created on first write — no "create namespace" endpoint.
- First query to a cold namespace is ~300ms (object storage). Subsequent: ~8ms (cache).
- Always truncate large MCP output: `.substring(0, 800)`. Cast before string methods: `row.field as string`.
- Only `filterable: true` attributes can appear in filters.
- Only prefix globs are efficient: `"tpuf*"` is fast, `"*tpuf*"` scans everything.
- `include_attributes: true` returns everything — prefer listing specific fields.
- Patches re-write the full row — avoid patching rows with large attributes frequently.
