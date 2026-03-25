# Write — Upsert, Patch, Delete, Configure Schema

Always check existing schema/metric before writing to an existing namespace:
```typescript
async function run(client) {
  const ns = client.namespace("NAMESPACE_NAME");
  const metadata = await ns.metadata();
  console.log(JSON.stringify(metadata, null, 2));
}
```

## Upsert documents

```typescript
async function run(client) {
  const ns = client.namespace("my-docs");
  const result = await ns.write({
    upsert_rows: [
      { id: 1, vector: [0.1, 0.2, 0.3], title: "Doc 1", category: "tech" },
      { id: 2, vector: [0.4, 0.5, 0.6], title: "Doc 2", category: "science" },
    ],
    distance_metric: "cosine_distance",
  });
  console.log(`Rows affected: ${result.rows_affected}`);
}
```

## Upsert with schema (first write)

Set schema explicitly to configure FTS, vector types, filterability:

```typescript
async function run(client) {
  const ns = client.namespace("articles");
  const result = await ns.write({
    upsert_rows: [
      { id: 1, vector: [0.1, 0.2], content: "article text", metadata: "extra" },
    ],
    distance_metric: "cosine_distance",
    schema: {
      content: { type: "string", full_text_search: true },
      vector: { type: "[768]f16", ann: true },
      metadata: { type: "string", filterable: false },
    },
  });
  console.log(`Rows affected: ${result.rows_affected}`);
}
```

## Patch (update attributes only)

```typescript
await ns.write({ patch_rows: [{ id: 1, category: "updated" }] })
```

## Delete by ID

```typescript
await ns.write({ deletes: [1, 2, 3] })
```

## Delete by filter

**Always confirm with user before executing.**

```typescript
await ns.write({ delete_by_filter: ["status", "Eq", "expired"] })
```

## Verify after write

Query back a sample to confirm:
```typescript
async function run(client) {
  const ns = client.namespace("NAMESPACE_NAME");
  const result = await ns.query({ rank_by: ["id", "asc"], limit: 3, include_attributes: true });
  console.log(JSON.stringify(result.rows, null, 2));
}
```

## Schema tips

- `full_text_search: true` — enable on text fields for BM25 search
- `[DIMS]f16` instead of `[DIMS]f32` — ~50% less storage, minimal quality loss
- `filterable: false` — 50% storage discount on attributes you won't filter on
- Schema auto-infers from first write if not set. **Prefer explicit** for FTS/vector config.

## Generating SDK code

If the user wants code (not a live write), use these patterns:
- **TypeScript**: `await ns.write({ upsert_rows: [...], distance_metric: "cosine_distance" })`
- **Python**: `ns.write(upsert_rows=[...], distance_metric="cosine_distance")`
- **Go**: `ns.Write(ctx, turbopuffer.NamespaceWriteParams{...})`

Use `search_docs` tool for full SDK reference.
