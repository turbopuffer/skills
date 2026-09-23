# Branching & Copying

**Branch**: an instant copy-on-write clone in the same region. Good for tests, dev sandboxes, and snapshots. Costs a flat $0.032.

```typescript
await tpuf.namespace("prod-test-1").branchFrom({ source_namespace: "prod" });
```

**Copy**: a full physical copy. Use it for backups, moving across regions, orgs, or clouds, resharding, or changing the encryption key. Up to 75% cheaper than re-upserting.

```typescript
await tpuf.namespace("prod-backup").copyFrom(
  { source_namespace: "prod", source_region: "aws-us-east-1" }, // region/api key optional
  { timeout: 30 * 60 * 1000 },
);
```

- The destination must be empty.
- Pinning isn't carried over. `read_only` is.
- To freeze a namespace, use `ns.updateMetadata({ read_only: true })`.
