# Pinning

By default, namespaces share query nodes and cache with other tenants. That's cheap, and caching keeps busy namespaces warm. Pinning reserves dedicated query nodes and SSD cache for one namespace. It stays hot, and latency and throughput are predictable.

## Cost — read before recommending

Pinning replaces per-query billing with **GB-hours**:

```
namespace size (GB) × replicas × hours pinned
```

- You pay for as long as it's pinned, whether or not it gets queries.
- Minimum billing is **128 GB** per replica and **10 minutes**. A 5 GB namespace is billed as 128 GB.
- Each replica multiplies the cost. Replicas are billed as soon as they're running, even before they're warm.
- It only pays off with **sustained** traffic. Break-even is usually around **10 QPS** on a large namespace. Below that, the default per-query billing is cheaper.

**Always confirm with the user before pinning or adding replicas.** Estimate the monthly cost first, e.g. 500 GB × 2 replicas × 730 h, and compare it against their current query bill.

## When to pin

Pin only if all of these hold:
- The namespace is large (**> 16 GB**).
- Traffic is **sustained** (**> 10 QPS**), not bursty.
- Cold queries or 429s from the 16-concurrent-query limit are hurting the product, or they want a predictable bill.

Don't pin:
- Small or per-tenant namespaces, or spiky/low traffic. Call `ns.hintCacheWarm()` when a user session starts instead.
- Just to fix occasional slow first queries. Warming handles that for free.

## Setup

```typescript
await ns.updateMetadata({ pinning: { replicas: 1 } }); // or `true` for 1 replica
await ns.updateMetadata({ pinning: null });            // unpin (billing stops)
```

- Takes up to ~30 minutes. Queries keep working in the meantime.
- Check status with `metadata.pinning.status`: `ready_replicas`, `replicas`, and `utilization`.
- Each replica serves roughly 100–1,000 QPS. Start with 1 and add one when `utilization` stays above 0.9 or queries return 429. Replicas don't autoscale.
- On sharded namespaces, each shard gets its own replicas (8 shards × 2 replicas = 16 nodes), but billing is still based on total size.
- Copies and branches start unpinned.
