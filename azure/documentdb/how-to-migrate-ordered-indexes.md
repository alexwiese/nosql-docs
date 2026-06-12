---
title: Migrate to ordered indexes in Azure DocumentDB
description: Learn how ordered indexes work, understand the key behavior changes, and follow a safe, reversible migration path from non-ordered to ordered indexes.
author: marcelofonseca
ms.author: marcrod
ms.topic: how-to
ms.date: 06/03/2026
---

# Migrate to ordered indexes in Azure DocumentDB

Ordered indexes deliver faster writes, smaller indexes, and pushdown-driven reads (sort, `$group`, and covered queries). The one behavior you must plan for: an ordered compound index only serves queries that filter on its first field. Before you migrate, audit your query patterns, add single-field indexes for any independently queried path, and roll out reversibly by hiding the old index before you drop it.

> [!NOTE]
> All clusters created on or after December 2025 have ordered indexes enabled by default. If your cluster predates that, ordered indexes aren't enabled by default - you must pass the `storageEngine` flag explicitly when creating indexes.

## Index types

Azure DocumentDB supports two categories of index.

**Non-ordered indexes (traditional)** were originally optimized for point lookup (OLTP) queries, where direct key access was the dominant workload. As complex queries, aggregations, and range-based access patterns became increasingly common on Azure DocumentDB, a more capable index architecture was needed. Non-ordered indexes don't support advanced MongoDB-style query semantics, and each indexed path keeps an independent index tree.

**Ordered indexes (enhanced)** come in three shapes:

- **Single-path indexes** - individual fields with ordering support.
- **Composite (compound) indexes** - multiple fields indexed together with ordering semantics.
- **Wildcard indexes** - now supported with ordered index features.

| Capability | Non-ordered | Ordered |
| --- | :--: | :--: |
| Point lookup (OLTP) optimization | Yes | Yes |
| Range query / sort optimization | Limited | Yes |
| Write throughput and latency | Baseline | Improved |
| MongoDB-style semantics (`$group`, `$sort`, covered) | No | Yes |
| Index tree per compound index | One per path | Single composite tree |
| Custom `storageEngine` option | No | `enableOrderedIndexes: true` |

## Benefits of ordered indexes

### Better write performance and reduced index bloat

Ordered indexes use a more efficient internal structure that improves both write throughput and write latency compared to non-ordered indexes. The downstream effects are:

- Reduced write contention at write time.
- Reduced write IOPS.
- Improved write throughput and lower write latency.
- Less index bloat over time.

### MongoDB-style query semantics

Ordered indexes unlock a class of optimizations that were previously unavailable:

- **`$group` operations** leverage index ordering to perform grouping.
- **`$sort` pushdown** happens at the index level, avoiding in-memory sorting.
- **Range queries** see materially better performance on range filters.
- **Covered index queries** return results entirely from index data, with no document reads required, when every path the query accesses is present in the index. This reduces I/O and improves latency.

### Optimized compound indexes

With non-ordered indexes, each field in a compound index has its own separate index tree. To satisfy a query that filters on multiple fields, the database must scan each tree independently and then **intersect the results** — computing the common set of matching documents across both trees.

With ordered indexes, all fields share a single composite tree. The database walks that one structure top-down and finds the matching documents directly — no intersection needed. The result is faster reads and lower resource usage.

## Understand the behavior change

This difference is the most important difference to understand before migrating. **With ordered compound indexes, a query must filter on the first index path to use the index.**

Given the index `{ a: 1, b: 1 }`:

| Query | Uses the ordered index? |
| --- | :--: |
| `find({ a: 5, b: 10 })` | Yes |
| `find({ a: 5 })` | Yes |
| `find({ b: 10 })` | No |

With *non-ordered* compound indexes, each path had an independent index tree, so an index on `{ a: 1, b: 1 }` could serve a query filtering only on `b`. With *ordered* compound indexes, all paths live in a single composite tree for performance - which means queries must start with the first indexed field.

> [!IMPORTANT]
> This change is where a migration silently degrades. A query that previously used your compound index on a secondary field stops using it. **Add a separate single-field index for each path that you query independently.**

```javascript
// To support filtering on b alone, maintain two indexes:
{ a: 1, b: 1 }   // queries on a (with or without b)
{ b: 1 }         // queries on b alone
```

## Identify ordered versus non-ordered indexes

List indexes by using `db.collection.getIndexes()` and check the options. An ordered index shows a custom `storageEngine` flag, while a traditional index shows nothing.

**Ordered index**

```json
{
  "v": 2,
  "key": { "field1": 1, "field2": -1 },
  "name": "idx_field1_field2",
  "storageEngine": {
    "enableOrderedIndexes": true
  }
}
```

**Non-ordered index**

```json
{
  "v": 2,
  "key": { "field1": 1, "field2": -1 },
  "name": "idx_field1_field2_old"
}
```

### Index key order matters

When you define compound indexes:

- Place **equality (more restrictive) filters first**.
- Put **higher cardinality paths before lower cardinality paths**.

This order maximizes index efficiency and selectivity.

```javascript
// status is low cardinality: ["active", "inactive"]
// userId is high cardinality: unique per user
// → place `userId` first
{ userId: 1, status: 1 }
```

### `$or` branches after an equality filter

When an `$or` follows an equality filter, make sure your index structure can satisfy both the equality condition and each `$or` condition. This setup often requires one compound index per branch.

```javascript
db.collection.find({
  a: 5,
  $or: [
    { b: 10 },
    { c: 20 }
  ]
})

// Consider index structures like:
{ a: 1, b: 1 }
{ a: 1, c: 1 }
```

## Sort pushdown rules

For a sort to be pushed down to the index (and avoid in-memory sorting), the index must be laid out so that walking it top-down already yields the desired order.

### Prefix match (with `$in` or non-equality operators)

The sort specification must match the **query filter prefix**.

```javascript
// Index: { a: 1, b: 1 }

// Sort pushdown — sort specification matches the index prefix
db.collection
  .find({ a: 1, b: { $in: ["1", "2", "3"] } })
  .sort({ a: 1, b: 1 });

// In-memory sort — sort specification starts with b, which is not the index prefix
db.collection
  .find({ a: 1, b: { $in: ["1", "2", "3"] } })
  .sort({ b: 1 });
```

### Composite index sort matching

To ensure maximum index utilization for sort operations, the sort specification must match both the **fields and their directions** as declared in the index — **or** be the **complete reverse** of the index.

Given the index `{ a: 1, b: -1 }`:

| Sort specification | Behavior |
| --- | --- |
| `.sort({ a: 1, b: -1 })` | Full index-based sort (forward scan) |
| `.sort({ a: -1, b: 1 })` | Full index-based sort (reverse scan) |
| `.sort({ a: 1, b: 1 })` | Partial — only `a` uses the index; `b` sorted in memory |
| `.sort({ a: 1 })` | Partial — only `a` uses the index |

> [!TIP]
> Sorts that don't fully match the index order still benefit from **partial (incremental) ordering** — the index handles what it can, and the database completes the remaining fields in memory.

### How BSON index terms are structured

Each term is a byte sequence that encodes metadata flags, such as whether a value is truncated, and the actual BSON value. When sorting, the database compares terms in this order:

1. **Path** — where in the document the value lives.
1. **Sort order** — ascending or descending.
1. **Value** — the actual data.
1. **Truncation** — truncated terms are considered greater when everything else is equal.

**Top-level arrays are skipped** during indexing, striking a balance between a leaner index and dependable sorting. This choice is justified because queries involving top-level arrays are uncommon.

> [!WARNING]
> **Multi-key limitation.** If an indexed path contains arrays, the index is *multi-key*, and features like `$group` and covered index queries **can't fully utilize it**. To check: run `explain`, inspect the index details, and if `isMultiKey: true`, those operations are restricted.

### Queries on array fields

When a query filters on fields inside an array, the index narrows the candidate documents and the database confirms each match against the actual documents. This behavior is expected. The index still provides a significant speedup by limiting how many documents need to be examined.

```javascript
// Index on nested array fields
db.orders.createIndex(
  { "results.product": 1, "results.score": 1 },
  { storageEngine: { enableOrderedIndexes: true } }
)

// Query that uses the index
db.orders.find({
  "results.product": "xyz",
  "results.score": { $gte: 8 }
})
```

### Operators and their pushdown behavior

| Operators | Index pushdown behavior |
| --- | --- |
| `$range` `$elemMatch` `$in` `$eq` `$gte` `$lte` `$gt` `$lt` | Often pushed to the index, sometimes with a runtime recheck. |
| `$type` `$nin` `$ne` | Trickier - these operators might require a full scan or can't be pushed at all. |
| `$all` `$size` `$exists` `$regex` | Some operators can be rewritten or partially supported by the index. |
| `$mod` `$bitsAnySet` `$bitsAllSet` `$bitsAnyClear` `$bitsAllClear` | Supported, but with caveats. |

## Evaluate before migrating

Before you modify a production index, audit your query patterns and index configuration to avoid silent regressions.

- **Check which fields your queries filter on independently.** If any query filters only on a non-first field of a compound index, add a separate single-field index for that path. The ordered compound index won't serve those queries on its own.
- **Check whether any index you plan to migrate is a unique index.** Creating a unique ordered index blocks all writes to the collection for the duration of the operation. Schedule this operation during a maintenance window or low-traffic period. Nonunique indexes can follow the standard migration steps in the next section.

> [!NOTE]
> As a general rule, always create unique indexes on an empty collection before loading data, to avoid the write-blocking build and reduce resource contention in large collections with heavy read-write traffic.

## Migrate to ordered indexes

The safe approach is reversible at every stage: create the new index, hide the old one, watch the metrics, and only drop the old index once latencies are stable.

**Example scenario**

Suppose you have an existing non-ordered compound index on a collection:

```javascript
// Existing non-ordered index
db.collection.createIndex({ a: 1, b: 1 }, { name: "idx_a_b" })
```

Your application runs three distinct query patterns against this collection:

```javascript
db.collection.find({ a: 5 })           // filters on a alone
db.collection.find({ b: 10 })          // filters on b alone
db.collection.find({ a: 5, b: 10 })    // filters on both fields
```

With a non-ordered index, all three queries can use the compound index because each field had its own independent index tree. With an ordered compound index, only queries that filter on the first field (`a`) are covered. The query on `b` alone no longer uses the index.

The migration plan creates two ordered indexes to restore full coverage:

```javascript
// Ordered compound index — covers queries on a, and a + b
db.collection.createIndex(
  { a: 1, b: 1 },
  { name: "idx_a_b_ordered", storageEngine: { enableOrderedIndexes: true } }
)

// Ordered single-field index — covers queries on b alone
db.collection.createIndex(
  { b: 1 },
  { name: "idx_b_ordered", storageEngine: { enableOrderedIndexes: true } }
)
```

### Step 1 — Check whether the index is already ordered

```javascript
db.testCollection.getIndexes()
```

If the target index already shows `storageEngine.enableOrderedIndexes: true`, it's ordered. If the flag is `false` or absent, it's not.

### Step 2 — Create the new ordered index

Based on your query patterns and the guidance in the preceding section, pass the `storageEngine` option to create the ordered index alongside the existing one. You don't need to drop the old index first - Azure DocumentDB allows multiple indexes on the same field or field combination, as long as they have different names.

```javascript
db.collection.createIndex(
  { field1: 1, field2: -1 },
  {
    name: "idx_field1_field2_ordered",
    storageEngine: { enableOrderedIndexes: true }
  }
)
```

After this step, both the old and the new index exist simultaneously. Queries continue to use whichever index the query planner selects, so there's no disruption. Don't drop the old index yet - you will hide it in the next step to safely validate the new one first.

### Step 3 — Hide the old index and verify the new one

**Before hiding — record the baseline**

Run `explain` on a critical query to record which index the planner selects and how many documents it examines:

```javascript
db.collection.find({ field1: "value" }).explain("executionStats")
```

While the old index is still visible, the output shows it in the winning plan along with actual execution counts:

```json
"winningPlan": {
  "stage": "FETCH",
  "inputStage": {
    "stage": "IXSCAN",
    "indexName": "idx_field1_field2_old",
    "isMultiKey": false
  }
},
"executionStats": {
  "nReturned": 10,
  "totalKeysExamined": 10,
  "totalDocsExamined": 10
}
```

**Hide the old index**

Use `collMod` with `hidden: true` to hide the old index without dropping it:

```javascript
db.runCommand({
  collMod: "collectionName",
  index: {
    name: "idx_field1_field2_old",
    hidden: true
  }
})
```

**After hiding — confirm the new index is selected**

Run the same query again. The planner should now select the new ordered index. A healthy result shows the same or lower `totalKeysExamined` and `totalDocsExamined` counts:

```json
"winningPlan": {
  "stage": "FETCH",
  "inputStage": {
    "stage": "IXSCAN",
    "indexName": "idx_field1_field2_ordered",
    "isMultiKey": false
  }
},
"executionStats": {
  "nReturned": 10,
  "totalKeysExamined": 10,
  "totalDocsExamined": 10
}
```

Monitor query latency and performance metrics for a period before proceeding. Run `explain` on all critical queries, not just one, to make sure every query pattern is covered. If you observe latency spikes or unexpected behavior, roll back by unhiding the old index:

```javascript
db.runCommand({
  collMod: "collectionName",
  index: {
    name: "idx_field1_field2_old",
    hidden: false
  }
})
```

### Step 4 — Review index usage with indexStats

Run `$indexStats` to see how many operations each index served. Record the `accesses.ops` value for the old index right after you hide it, then recheck after a few days. If the count hasn't changed, the old index is no longer used.

```javascript
db.collection.aggregate([{ $indexStats: {} }])
```

The output includes one entry per index. Look for the old index by name and compare the `ops` count over time:

```json
{
  "name": "idx_field1_field2_old",
  "accesses": {
    "ops": 0,
    "since": "2026-06-01T00:00:00.000Z"
  }
}
```

### Step 5 — Drop the old index

When query latencies stabilize and the old index shows no recent accesses, drop it.

```javascript
db.collection.dropIndex("idx_field1_field2_old")
```

## Check whether ordered indexes are enabled by default

Create a test index on an empty collection, and then inspect it.

```javascript
db.testCollection.createIndex({ testField: 1 })
db.testCollection.getIndexes()
```

If the output includes the `storageEngine` option with `enableOrderedIndexes: true`, ordered indexes are on by default. If the option is absent, you need to pass the flag explicitly when creating indexes.

## Related content

- [Manage indexing in Azure DocumentDB](indexing.md)
- [Indexing best practices in Azure DocumentDB](how-to-create-indexes.md)
- [Background indexing in Azure DocumentDB](background-indexing.md)
- [Indexing scenarios in Azure DocumentDB](how-to-index.md)
