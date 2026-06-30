---
title: Read query explain output in Azure DocumentDB
description: Learn how to run the explain command and interpret the output to diagnose query performance issues in Azure DocumentDB.
author: fonsecamar
ms.author: marcrod
ms.topic: how-to
ms.date: 06/22/2026
ai-usage: ai-assisted
---

# Read query explain output in Azure DocumentDB

The `explain()` command shows how Azure DocumentDB executes a query. Use `explain()` to identify whether a query uses an index, how many documents are scanned, and where performance bottlenecks occur.

## Run explain

Run `explain()` on any `find` or `aggregate` command by appending it to the query.

```javascript
db.collection.find({ field: "value" }).explain("executionStats")
```

Use one of three verbosity modes:

| Mode | Description |
| --- | --- |
| `"queryPlanner"` | Returns the winning query plan without running the query. |
| `"executionStats"` | Runs the query and returns execution statistics for the winning plan. |
| `"allPlansExecution"` | Runs the query and returns statistics for all candidate plans. |

For performance analysis, use `"executionStats"` as a starting point.

## Understand the output structure

The explain output is divided into two main sections.

### queryPlanner

The `queryPlanner` section describes how the query engine plans to execute the query before running it.

```json
"queryPlanner": {
  "winningPlan": {
    "stage": "FETCH",
    "inputStage": {
      "stage": "IXSCAN",
      "indexFilterSet": [
        { "$eq": { "status": "1" } }
      ],
      "indexName": "status_1"
    }
  }
}
```

Key fields in `queryPlanner`:

| Field | Description |
| --- | --- |
| `winningPlan` | The query plan selected by the optimizer. |
| `winningPlan.stage` | The execution stage. Common values are `COLLSCAN`, `IXSCAN`, and `FETCH`. |
| `winningPlan.inputStage` | The child stage that feeds data into the current stage. |
| `rejectedPlans` | Other candidate plans that the optimizer didn't select. |

### executionStats

The `executionStats` section contains metrics from the actual query execution.

```json
"executionStats": {
  "nReturned": 10,
  "totalKeysExamined": 10,
  "totalDocsExamined": 10,
  "executionTimeMillis": 3
}
```

Key fields in `executionStats`:

| Field | Description |
| --- | --- |
| `nReturned` | Number of documents returned to the client. |
| `totalKeysExamined` | Number of index entries scanned. |
| `totalDocsExamined` | Number of documents scanned from the collection. |
| `executionTimeMillis` | Total time to execute the query, in milliseconds. |

## Identify execution stages

Each stage in the `winningPlan` represents an operation in the query pipeline. Identify the stages to understand how data flows through the execution plan.

| Stage | Description |
| --- | --- |
| `COLLSCAN` | Full collection scan. The engine reads all documents. This stage occurs when no suitable index exists. |
| `IXSCAN` | Index scan. The engine reads index entries to find matching documents. |
| `FETCH` | Document fetch. The engine loads full documents from storage after scanning an index. |
| `SORT` | In-memory sort. This stage occurs when no index supports the sort order. |
| `COUNT_SCAN` | Count using an index. This stage avoids loading documents to satisfy a count operation. |
| `COUNT` | In-memory count. The engine counts documents after loading them from the collection. |
| `GROUP` | In-memory grouping operation, typically produced by a `$group` aggregation stage. |
| `LIMIT` | Limits the number of documents passed to the next stage. |
| `DISTINCT_SCAN` | Scans an index to return distinct values for a field, skipping duplicate index entries. |
| `DISTINCT_UNWIND` | Unwinds array values during a distinct scan to enumerate each element separately. |
| `UNIQUE` | Deduplicates documents from the input stage, used in distinct operations. |
| `PARALLEL_MERGE` | Merges results from multiple parallel execution branches into a single stream. |

## Detect performance problems

Use the following patterns to identify common performance problems in the explain output.

### Collection scan instead of index scan

A `COLLSCAN` stage means the query reads every document in the collection. This operation is expensive for large collections.

**Indicator:** `winningPlan.stage` is `"COLLSCAN"`.

```json
"winningPlan": {
  "stage": "COLLSCAN"
}
```

**Action:** Create an index on the fields used in the query filter. If the query filters on multiple fields, sorts results, or uses aggregation stages such as `$group` or `$lookup`, create a compound index that covers all relevant fields.

```javascript
// Single field
db.collection.createIndex({ field: 1 })

// Multiple fields (filter + sort)
db.collection.createIndex({ filterField: 1, sortField: 1 })

// Aggregation pipeline with $match on multiple fields
db.collection.createIndex({ status: 1, createdAt: -1 })
```

### High ratio of documents examined to documents returned

Compare `totalDocsExamined` to `nReturned`. A large difference means the query scans many documents but returns few.

**Indicator:** `totalDocsExamined` is significantly higher than `nReturned`.

```json
"nReturned": 5,
"totalDocsExamined": 50000
```

**Action:** Add a more selective index or refine the query filter to reduce the scan range.

### In-memory sort

A `SORT` stage after a `FETCH` or `COLLSCAN` stage means the sort operation is performed in memory after loading documents. In-memory sorts consume memory and slow down queries at scale.

**Indicator:** A `SORT` stage appears in the plan.

```json
"winningPlan": {
  "stage": "SORT",
  "inputStage": {
    "stage": "COLLSCAN"
  }
}
```

**Action:** Create an index that includes the sort field and follows the Equality, Sort, Range (ESR) rule for compound indexes.

```javascript
db.collection.createIndex({ filterField: 1, sortField: 1 })
```

### Slow execution time

A high `executionTimeMillis` value indicates a slow query.

**Indicator:** `executionTimeMillis` is higher than expected for the number of documents returned.

```json
"executionTimeMillis": 8500,
"nReturned": 20
```

**Action:** Check for `COLLSCAN`, high `totalDocsExamined`, or in-memory `SORT` stages and address them with appropriate indexes.

## Interpret a well-optimized query

A well-optimized query uses an index, scans only the necessary keys, and returns documents with no in-memory sort.

```json
"executionStats": {
  "nReturned": 10,
  "totalKeysExamined": 10,
  "totalDocsExamined": 10,
  "executionTimeMillis": 2,
  "winningPlan": {
    "stage": "FETCH",
    "inputStage": {
      "stage": "IXSCAN",
      "indexName": "status_1_createdAt_1"
    }
  }
}
```

In this example:
- `totalKeysExamined` equals `nReturned`, which means the index is selective.
- `totalDocsExamined` equals `nReturned`, which means no extra documents are fetched.
- The plan uses `IXSCAN` followed by `FETCH`, which is the expected pattern for indexed queries.

## Run explain for aggregate pipelines

Run `explain()` on aggregate pipelines to inspect the execution plan for each stage.

```javascript
db.collection.explain("executionStats").aggregate([
  { $match: { status: "active" } },
  { $sort: { createdAt: -1 } },
  { $limit: 10 }
])
```

The output includes a `stages` array that shows how each pipeline stage maps to an internal execution plan. Inspect the first stage for `COLLSCAN` or `IXSCAN` to confirm whether the `$match` filter uses an index.

## Related content

- [Indexing best practices](how-to-create-indexes.md)
- [Indexing scenarios](how-to-index.md)
- [Tune query performance with Index Advisor](index-advisor.md)
- [Wildcard indexes](how-to-create-wildcard-indexes.md)
