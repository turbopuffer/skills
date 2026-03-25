# Bulk Ingestion — Large-Scale Data Loading

## Key constraints

- **512 MB max batch size** — per upsert request
- **Bigger batches are better** — larger batches improve throughput and get up to 50% batch discount on write costs
- **Writes are immediately searchable** with strong consistency (default). Eventual consistency queries may be up to 60s stale.
- **2 GB unindexed data limit** — by default, exceeding this triggers 429 backpressure on writes (see `disable_backpressure` below)

## Recommended pattern

### 1. Use large batches

Pack as many documents as possible per write call (up to 512MB). Larger batches = better throughput and lower cost.

```typescript
async function run(client) {
  const ns = client.namespace("my-namespace");
  const batch = [];
  for (let i = 0; i < 10000; i++) {
    batch.push({ id: i, vector: [...], text: "..." });
  }
  const result = await ns.write({
    upsert_rows: batch,
    distance_metric: "cosine_distance",
  });
  console.log(`Wrote ${result.rows_affected} rows in ${result.performance?.server_total_ms}ms`);
}
```

### 2. Use concurrent writes

Especially for single-threaded runtimes (Node.js, Python), use multiple processes to write batches in parallel. Upserting is generally bottlenecked by serialization and compression on the client side.

### 3. Monitor indexing progress

```typescript
async function run(client) {
  const ns = client.namespace("my-namespace");
  const metadata = await ns.metadata();
  console.log("Index status:", metadata.index?.status);
  console.log("Approx rows:", metadata.approx_row_count);
  // Check exhaustive search count for unindexed backlog
  const result = await ns.query({ rank_by: ["id", "asc"], limit: 1 });
  console.log("Unindexed docs:", result.performance?.exhaustive_search_count);
}
```

### 4. Set schema on first write

Don't let schema auto-infer — set it explicitly on the first batch:

```typescript
await ns.write({
  upsert_rows: firstBatch,
  distance_metric: "cosine_distance",
  schema: {
    content: { type: "string", full_text_search: true },
    vector: { type: "[768]f16", ann: true },
    metadata: { type: "string", filterable: false },
  },
});
```

## Column format

`upsert_columns` is an alternative format that may be more natural for columnar data:

```typescript
await ns.write({
  upsert_columns: {
    id: [1, 2, 3],
    vector: [[0.1, 0.2], [0.3, 0.4], [0.5, 0.6]],
    title: ["Doc 1", "Doc 2", "Doc 3"],
  },
  distance_metric: "cosine_distance",
});
```

## Disable backpressure for large ingestions

By default, writes return 429 when unindexed data exceeds 2 GB. For initial data loading or bulk updates, you can disable this with `disable_backpressure: true`:

```typescript
await ns.write({
  upsert_rows: largeBatch,
  distance_metric: "cosine_distance",
  disable_backpressure: true,
});
```

**Important tradeoffs when backpressure is disabled:**
- **Strongly consistent queries will return errors** while unindexed data exceeds 2 GB — you must use eventual consistency (`consistency: { level: "eventual" }`) during ingestion
- **Eventually consistent queries only search the first 128 MB of unindexed data** — so query results will be incomplete until indexing catches up
- **`approx_row_count` and `approx_logical_bytes` in metadata won't update** until all data is indexed
- **Track indexing progress** via `unindexed_bytes` in the metadata endpoint's `index` field

**Best practice**: Use `disable_backpressure` for the initial bulk load, then switch back to normal writes (without the flag) once the bulk load is complete and indexing has caught up.

## Parallel ingestion across namespaces

If you need higher aggregate throughput, shard across multiple namespaces and write in parallel. Each namespace processes writes independently.

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| 429 on writes | Unindexed backlog > 2GB | Slow down; wait for indexing to catch up |
| p95 latency > 50s on upserts | Indexing pressure from sustained high write volume | Monitor `exhaustive_search_count`; let indexing catch up |
| Index status stuck at "indexing" | Large backlog processing | Wait; check `exhaustive_search_count` to monitor progress |
| Queries return stale data after write | Indexing delay | Use strong consistency (default) and wait for indexing |
