---
title: "Postgres Killed Our Query. The Fix Was One Index — Found in 30 Seconds."
description: "An endpoint that was fast in staging fell over in production. EXPLAIN ANALYZE showed a sequential scan on a 2M-row table. The index trade-off, and why I now check query plans before blaming the ORM."
pubDate: 2026-01-22
tags: [postgresql, performance, dotnet, ef-core]
draft: false
---

The endpoint ran in 40ms in staging and 9 seconds in production. Same code, same EF Core queries, same database engine. The only difference was the amount of data — and the query plan that amount of data forced.

## Seeing the truth

EF Core generates the SQL, but Postgres decides how to run it. So the first move is always the same: take the generated SQL and ask the database what it thinks.

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT o.* FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE c.region = 'APAC'
  AND o.created_at >= now() - interval '30 days'
ORDER BY o.created_at DESC
LIMIT 50;
```

The plan told the whole story in one line:

```
Seq Scan on orders  (cost=0.00..387421.00 rows=1900000) (actual time=0.041..9123.220 ...)
  Filter: (created_at >= ...)
  Rows Removed by Filter: 1980000
```

A sequential scan over two million rows, discarding 99% of them, on every request.

## The index

```sql
CREATE INDEX CONCURRENTLY idx_orders_created_at_region
  ON orders (created_at DESC)
  INCLUDE (customer_id)
  WHERE created_at >= now() - interval '90 days';
```

A partial index — recent orders only — because the query pattern never touches older data, and a partial index stays small enough to live entirely in cache. The composite `INCLUDE` column lets Postgres serve the join key from the index itself. Result: 9 seconds → 60 milliseconds, and the planner switched to an index scan with a bitmap heap read of only the matching rows.

## The trade-off nobody mentions in the tutorial

Indexes aren't free: this one costs write throughput on `orders` and ~40MB of storage. For a hot endpoint serving thousands of requests per minute, that's an excellent trade. For a table that's write-heavy and queried rarely, it wouldn't be. **The decision isn't "add an index?" — it's "does this read pattern justify the write tax?"**

## The habit this built

1. `EXPLAIN (ANALYZE, BUFFERS)` before touching code — the plan names the crime
2. Suspicious of ORMs only in one direction: not for the SQL they generate, but for the **indexes nobody added** because the generated code "just worked" on ten test rows
3. `CONCURRENTLY` always in production — a plain `CREATE INDEX` takes a lock that blocks writes, which is its own outage story

Nine seconds to 60ms, one statement, thirty seconds of diagnosis. The most satisfying line I wrote that quarter wasn't application code at all.
