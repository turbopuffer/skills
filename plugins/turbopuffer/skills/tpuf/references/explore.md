# Explore — List Namespaces, Inspect Schema & Metadata

## List namespaces

```typescript
async function run(client) {
  for await (const ns of client.namespaces()) {
    console.log(ns.id);
  }
}
```

With prefix filter:
```typescript
async function run(client) {
  for await (const ns of client.namespaces({ prefix: "prod-" })) {
    console.log(ns.id);
  }
}
```

## Describe a namespace

```typescript
async function run(client) {
  const ns = client.namespace("NAMESPACE_NAME");
  const metadata = await ns.metadata();
  console.log(JSON.stringify(metadata, null, 2));
}
```

### Present as

```
NAMESPACE_NAME — ~125K rows, 89MB
| Attribute | Type     | Filterable | FTS | ANN |
|-----------|----------|------------|-----|-----|
| vector    | [768]f32 | -          | -   | yes |
| title     | string   | yes        | -   | -   |
| content   | string   | no         | yes | -   |
Distance metric: cosine_distance | Index: up-to-date
Created: 2024-11-02 | Updated: 2025-03-24
```

## What to suggest next

- Has vectors → load `references/query.md` for vector search
- Has FTS attributes → load `references/query.md` for full-text search
- Empty/doesn't exist → load `references/write.md` to add data
- Schema looks wrong → load `references/doctor.md` to audit
