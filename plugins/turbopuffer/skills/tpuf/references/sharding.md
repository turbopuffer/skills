# Sharding

Splits one namespace across internal shards. Clients still use a single namespace, and turbopuffer fans queries out and merges the results.

Use it only when a namespace would exceed **1 TB or ~500M docs per shard**. For multi-tenancy, use one namespace per tenant. For more QPS, use pinning.

```typescript
await ns.write({
  upsert_rows: firstBatch,
  distance_metric: "cosine_distance",
  sharding: { num_shards: 4 }, // first write only; 1–256
});
```

- Size it as `num_shards ≈ ceil(expected TB)`, with room to grow. Over-sharding raises tail latency.
- The shard count can't change. To reshard, copy into a new namespace:
  `await tpuf.namespace("docs-v2").write({ copy_from_namespace: "docs", sharding: { num_shards: 16 } })`
- There's no extra cost per shard. Branching isn't supported on sharded namespaces yet.
- Eventually consistent reads can differ between shards.
