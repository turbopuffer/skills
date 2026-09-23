# Bulk Ingestion & Indexing

Writes land in a log right away and are indexed in the background. When more than **2 GB** is waiting to be indexed, writes return 429.

## Loading fast

1. Set the schema, and `sharding` if the namespace will pass 1 TB, on the first write.
2. Use large batches (up to 512 MB) with `upsert_columns`.
3. Run many writers in parallel (~2× CPUs, as separate processes in Python/Node).
4. For backfills, add `disable_backpressure: true` and query with `consistency: { level: "eventual" }` until indexing catches up. This works for upserts and deletes only.
5. Mark attributes you never filter on `filterable: false` so there's less to index.

## Checking the indexing backlog

```bash
curl -s "https://$TURBOPUFFER_REGION.turbopuffer.com/v1/namespaces/$NS/metadata" \
  -H "Authorization: Bearer $TURBOPUFFER_API_KEY" | jq .index
```

- `up-to-date`: done.
- `updating` with `unindexed_bytes` going down: catching up.
- Not going down with no new writes: contact turbopuffer support.

Row counts don't update during a `disable_backpressure` backfill.
