# 429 Errors

| Cause | Fix |
|---|---|
| **Query concurrency**: more than 16 queries in flight on one unpinned namespace | Make queries faster, use eventual consistency, warm the cache, cap concurrency in the client, or pin with replicas |
| **Pinned namespace saturated** | Add replicas when `pinning.status.utilization` > 0.9 |
| **Write backpressure**: more than 2 GB waiting to be indexed | `disable_backpressure: true` for backfills, or slow down. See `references/ingestion.md` |
| **Embedding rate limit** | Back off, send smaller batches in parallel, or ask turbopuffer to raise the limit |

- The limit counts concurrent queries, not QPS: 10ms queries still allow ~1,600 QPS.
- kNN queries count as 2 and aggregations as 4 toward the limit.
- The SDKs retry 429s automatically.
