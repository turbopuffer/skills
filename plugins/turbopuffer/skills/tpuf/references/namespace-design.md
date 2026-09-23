# Namespace Design

Make namespaces as small as possible without routinely querying more than one at a time. Namespaces are unlimited and free to create.

| Situation | Do |
|---|---|
| Tenants never search each other's data | One namespace per tenant |
| Different schemas / doc types | Separate namespaces |
| Shared corpus with per-user access | ACL arrays on docs + filter (below) |
| One corpus > 1 TB / 500M docs | `references/sharding.md` |
| Large namespace, high sustained QPS | `references/pinning.md` |

## Permissions

There's no built-in row-level access control. Store who can read each doc and filter on it:

```typescript
filters: ["Or", [["groups", "ContainsAny", user.groups], ["user_ids", "Contains", user.id], ["is_public", "Eq", true]]]
```

Mark public docs with a boolean. Don't rely on empty arrays. Declare ID arrays as `[]uuid`.

## Key limits

- 500M docs / 1 TB per shard, up to 256 shards
- 16 concurrent queries per unpinned namespace
- 10k writes/s per namespace
- 512 MB per write
- 8 vector columns and 1,024 attributes per namespace

Names must match `[A-Za-z0-9-_.]{1,128}`.
