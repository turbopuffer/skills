# Write

```typescript
// Upsert (replaces whole docs). Use upsert_columns for bulk.
await ns.write({ upsert_rows: [{ id: 1, vector: v, title: "Doc" }], distance_metric: "cosine_distance" });

// Patch (not vectors; missing IDs are ignored)
await ns.write({ patch_rows: [{ id: 1, title: "New" }] });

// Delete
await ns.write({ deletes: [1, 2] });
await ns.write({ delete_by_filter: ["status", "Eq", "expired"] }); // confirm first
await ns.write({ patch_by_filter: { filters: ["status", "Eq", "draft"], patch: { status: "live" } } });

// Conditional: only write if newer
await ns.write({ upsert_rows: docs, upsert_condition: ["version", "Lt", { $ref_new: "version" }] });
```

`delete_by_filter` handles 5M rows per request and `patch_by_filter` 50k. If `rows_remaining` is true, repeat the request.

## Schema

Types are inferred on first write. Declare explicitly when needed:

```typescript
schema: {
  id: "uuid",
  vector: { type: "[768]f16", ann: true },         // f16 / i8 are cheaper than f32
  content: { type: "string", full_text_search: true },
  raw: { type: "string", filterable: false },     // 50% cheaper if never filtered
}
```

Schema-only writes (`ns.write({ schema })`) can toggle `filterable`, `full_text_search`, `regex`, `glob`, `fuzzy`, and `embed`. Queries return 409 until a new index is built. Anything else (types, dims, distance metric) requires a new namespace.
