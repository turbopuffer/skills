# Troubleshoot 429 Errors

## Step 1: Identify the 429 type

Check the error message — there are two distinct causes:

| Error message | Cause | Section |
|--------------|-------|---------|
| `"Too many concurrent queries to a single namespace"` | Query concurrency limit hit | [Query 429s](#query-429s) |
| 429 on write/upsert requests | Indexing backpressure — unindexed data > 2GB | [Write 429s](#write-429s) |

---

## Query 429s

### How the limit works

- **16 concurrent queries per namespace** (server-side semaphore)
- The 17th query waits up to 800ms for a slot
- If no slot opens in 800ms → HTTP 429
- This is a **concurrency** limit, not a QPS limit — effective QPS depends on query latency:

| Query latency | Effective max QPS |
|--------------|-------------------|
| 10ms | ~1600 QPS |
| 50ms | ~300 QPS |
| 200ms | ~80 QPS |

- **Multi-query**: each sub-query within a multi-query request counts against the limit
- **SDKs retry automatically** with exponential backoff on 429s

### Step 2: Diagnose

```typescript
async function run(client) {
  const ns = client.namespace("NAMESPACE_NAME");
  const result = await ns.query({ rank_by: ["id", "asc"], limit: 1 });
  console.log("Latency:", result.performance?.server_total_ms, "ms");
  console.log("Cache:", result.performance?.cache_temperature);
  console.log("Execution:", result.performance?.query_execution_ms, "ms");
}
```

Ask:
- How many concurrent queries is the application sending?
- What's the query latency? (slow queries hold slots longer)
- Is the cache cold? (cold queries are much slower → fewer effective QPS)

### Step 3: Fix

In order of preference:

1. **Make queries faster** — reduce `limit`, use fewer `include_attributes`, avoid expensive filters (`Glob "*x*"`, large `In`). Faster queries free up slots sooner.
2. **Use eventual consistency** — `consistency: { level: "eventual" }`
3. **Add client-side concurrency control** — cap concurrent requests with a semaphore (e.g., 12 per namespace) to avoid hitting the server-side limit.

### Gotchas

- **Concurrency ≠ QPS.** Customers often confuse the 16-concurrent limit with a 16 QPS cap. With 10ms queries, 16 concurrent slots supports 1600+ QPS.
- **No `retry-after` header on query 429s.** The SDKs handle this with built-in exponential backoff. If using raw HTTP, implement your own backoff.
- **Multi-query consistency must be top-level.** `consistency` set per-query inside the `queries` array is silently ignored — defaults to strong, which is slower.

---

## Write 429s

### How backpressure works

- Writes are durable immediately but indexed asynchronously
- If unindexed data exceeds **2 GB**, writes return 429
- This protects query performance — large unindexed backlogs slow down every query via exhaustive WAL scan

### Step 2: Diagnose

```typescript
async function run(client) {
  const ns = client.namespace("NAMESPACE_NAME");
  const metadata = await ns.metadata();
  console.log("Index status:", metadata.index?.status);
  console.log("Unindexed bytes:", metadata.index?.unindexed_bytes);
}
```

### Step 3: Fix

1. **Use `disable_backpressure: true`** on write requests — the recommended approach for bulk loads. Writes continue without 429s regardless of backlog size.
   - **Tradeoff**: strongly consistent queries will error while unindexed > 2GB
   - Must use eventual consistency during ingestion: `consistency: { level: "eventual" }`
   - Eventually consistent queries only search the first 128 MB of unindexed data
   - Track progress via `unindexed_bytes` in the metadata `index` field

2. **Monitor the backlog** — check `unindexed_bytes` over time. If it's decreasing, indexing is catching up. If stable or growing, writes are outpacing indexing.

3. **Spread across namespaces** — if possible, shard data across multiple namespaces to distribute the indexing load. Each namespace indexes independently.

---

## Thresholds

| Metric | Default | Can be raised? |
|--------|---------|----------------|
| Concurrent queries/namespace | 16 | Yes — read replicas or concurrency bump (contact turbopuffer) |
| Max unindexed data before 429 | 2 GB | No — use `disable_backpressure` to bypass |
| Max queries in multi-query | 16 | No |
| 429 wait timeout | 800ms | No |

## Checklist

- [ ] Identify 429 type from error message (query vs write)
- [ ] If query 429: check query latency, cache temperature, concurrent request count
- [ ] If query 429: try eventual consistency first (biggest single improvement)
- [ ] If query 429 persists: request read replicas or concurrency bump
- [ ] If write 429: check `unindexed_bytes` — is backlog growing?
- [ ] If write 429: use `disable_backpressure: true` + eventual consistency
- [ ] Verify SDKs are retrying automatically (built-in exponential backoff)
