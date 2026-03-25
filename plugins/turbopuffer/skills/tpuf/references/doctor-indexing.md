# Doctor: Indexing — Diagnose Stuck Indexing, Backlog, Throttling

## How indexing works

Written data is asynchronously indexed for efficient retrieval. Unindexed data is still searchable via exhaustive scan of the write-ahead log — indexing affects search *performance*, not *correctness*. Writes are immediately visible with strong consistency (default).

## Step 1: Check indexing status

```typescript
async function run(client) {
  const ns = client.namespace("NAMESPACE_NAME");
  const metadata = await ns.metadata();
  console.log("Index status:", metadata.index?.status);
  console.log("Unindexed bytes:", metadata.index?.unindexed_bytes);
  console.log("Row count:", metadata.approx_row_count);
}
```

| `index.status` | `unindexed_bytes` | Meaning |
|----------------|-------------------|---------|
| `up-to-date` | 0 or small | Healthy — all data indexed |
| `updating` | < 2 GB | Normal — indexing in progress, will catch up |
| `updating` | > 2 GB | Backpressure zone — writes get 429 unless `disable_backpressure` is set |
| `updating` | > 10 GB | Large backlog — may take hours to fully index |
| `updating` | Not decreasing over time | Stuck — contact turbopuffer support |

## Step 2: Determine if indexing is progressing

Run the status check multiple times over a few minutes. If `unindexed_bytes` is:
- **Decreasing** → indexing is working, just slow. Normal after bulk writes.
- **Stable or increasing** → writes are outpacing indexing, or the indexer is stuck.

## Step 3: Check query-side impact

```typescript
async function run(client) {
  const ns = client.namespace("NAMESPACE_NAME");
  const result = await ns.query({ rank_by: ["id", "asc"], limit: 1 });
  console.log("Exhaustive search count:", result.performance?.exhaustive_search_count);
  console.log("Latency:", result.performance?.server_total_ms, "ms");
}
```

`exhaustive_search_count` shows how many unindexed docs are scanned per query. High values mean queries are slower because they're exhaustively scanning the WAL backlog.

## Gotchas

- **Bulk patching generates heavy indexing work.** Patches re-read all attributes of affected rows, then re-write. Patching millions of rows can produce tens of GB of unindexed data. Use `disable_backpressure: true` and eventual consistency during bulk patches.
- **`approx_row_count` and `approx_logical_bytes` don't update during `disable_backpressure` ingestion.** Use `unindexed_bytes` in the metadata `index` field to track progress instead.

## Checklist

- [ ] Check `index.status` and `unindexed_bytes`
- [ ] If `unindexed_bytes` > 2GB: is `disable_backpressure` set? If not, writes will 429
- [ ] If `unindexed_bytes` not decreasing over minutes: contact turbopuffer support
- [ ] Check `exhaustive_search_count` — high values explain slow queries during indexing
- [ ] If bulk patching: use `disable_backpressure: true` + eventual consistency
