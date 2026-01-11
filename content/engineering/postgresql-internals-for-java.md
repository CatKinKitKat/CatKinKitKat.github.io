+++
title = "PostgreSQL Internals Every Java Developer Should Know"
date = 2025-11-01
[taxonomies]
tags = ["postgresql", "database", "java"]
+++

Most Java developers interact with PostgreSQL through JPA and treat it as a black box. Write entity, call save, magic happens. I was that developer until a production incident taught me that understanding what happens below the ORM makes the difference between a system that scales and one that falls over at 3 AM.

## The WAL and Why It Matters

The Write-Ahead Log (WAL) is PostgreSQL's transaction log. Every data modification is written to the WAL before it's applied to the actual tables. This guarantees durability: if the server crashes after a WAL write but before the table write, PostgreSQL replays the WAL on startup and recovers the data.

Why you care as a Java developer:

- **Replication** depends on the WAL. Streaming replication sends WAL segments to replicas.
- **CDC tools** (Debezium) read the WAL via logical replication.
- **Disk usage** is affected by WAL retention. Long-running transactions keep WAL segments from being recycled.
- **Write performance** is bounded by WAL write speed. If your WAL is on slow storage, every write is slow.

The practical tip: put the WAL on fast storage (NVMe). If you're on cloud, use provisioned IOPS. This single infrastructure decision affects every write operation.

## Replication Slots and Logical Replication

Streaming replication (physical) copies the raw WAL bytes to a replica. The replica is an exact copy of the primary, including all databases and schemas.

Logical replication is more selective. It decodes the WAL into a structured stream of changes and can replicate specific tables to specific targets. This is what Debezium uses.

```sql
-- Create a logical replication slot
SELECT * FROM pg_create_logical_replication_slot('my_slot', 'pgoutput');

-- Create a publication
CREATE PUBLICATION my_pub FOR TABLE orders, customers;
```

The critical thing: replication slots retain WAL. If nothing consumes the slot, WAL accumulates until the disk is full. Always monitor `pg_replication_slots`:

```sql
SELECT slot_name, active,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained_wal
FROM pg_replication_slots;
```

If `active` is false and `retained_wal` is growing, something is wrong. Either the consumer died or it's not keeping up.

## HOT Updates: The Performance Feature Nobody Talks About

Heap-Only Tuples (HOT) is a PostgreSQL optimization for UPDATE-heavy workloads. When you update a row and the modified columns are not part of any index, PostgreSQL can store the new row version on the same page as the old one without updating any indexes.

Why this matters: without HOT, every UPDATE creates a new tuple AND updates every index on the table. For a table with 10 indexes, an update to one non-indexed column causes 10 index modifications. With HOT, those 10 index modifications are avoided.

How to enable it (it's automatic, but you can help):

1. **Leave fill factor room.** Set `fillfactor` below 100 to leave space on each page for HOT updates:

```sql
ALTER TABLE orders SET (fillfactor = 80);
```

This tells PostgreSQL to fill each 8KB page to 80%, leaving 20% for in-place updates.

2. **Don't index columns you update frequently.** An index on `updated_at` prevents HOT updates on that table. If you need the index, accept the trade-off. If you don't, remove it.

Monitor HOT update effectiveness:

```sql
SELECT relname,
       n_tup_upd AS updates,
       n_tup_hot_upd AS hot_updates,
       CASE WHEN n_tup_upd > 0
            THEN round(100.0 * n_tup_hot_upd / n_tup_upd, 1)
            ELSE 0
       END AS hot_pct
FROM pg_stat_user_tables
ORDER BY n_tup_upd DESC;
```

If `hot_pct` is low on a frequently updated table, check which indexes are preventing HOT updates.

## Index Types and When to Use Each

### B-tree (Default)

The workhorse. Supports equality and range queries (`=`, `<`, `>`, `BETWEEN`, `IN`). Use it for primary keys, foreign keys, and most WHERE clauses.

```sql
CREATE INDEX idx_orders_status ON orders (status);
```

### Hash

Equality only. Slightly faster than B-tree for pure equality lookups, but doesn't support range queries. Before PostgreSQL 10, hash indexes weren't WAL-logged and were unsafe. They're fine now, but I rarely use them because B-tree handles equality well enough.

### GIN (Generalized Inverted Index)

For containment queries: full-text search, JSONB, arrays.

```sql
CREATE INDEX idx_orders_tags ON orders USING gin (tags);
-- Now: SELECT * FROM orders WHERE tags @> '{"priority": "high"}';
```

GIN indexes are slow to update but fast to query. Use them for columns that are read-heavy and write-light.

### GiST (Generalized Search Tree)

For geometric data, range types, and full-text search (alternative to GIN). If you're doing PostGIS spatial queries, you're using GiST.

### BRIN (Block Range Index)

For very large tables where the data is physically ordered. BRIN stores min/max values per block range, making it tiny but effective for range queries on naturally ordered data.

```sql
-- Perfect for a time-series table where rows are inserted in order
CREATE INDEX idx_events_ts ON events USING brin (created_at);
```

BRIN indexes are orders of magnitude smaller than B-tree indexes on large tables. If your table has 100 million rows ordered by timestamp, a B-tree index might be 2GB while a BRIN index is 50KB.

### Index Selectivity

An index is useful only if it's selective - if it significantly reduces the number of rows the database needs to examine.

```sql
-- Bad: status has 3 distinct values across 1 million rows. Low selectivity.
CREATE INDEX idx_orders_status ON orders (status);
-- PostgreSQL will likely ignore this index and do a sequential scan.

-- Good: email is unique or near-unique. High selectivity.
CREATE INDEX idx_users_email ON users (email);
```

Check selectivity with:

```sql
SELECT attname,
       n_distinct,
       most_common_vals,
       most_common_freqs
FROM pg_stats
WHERE tablename = 'orders' AND attname = 'status';
```

If `n_distinct` is small relative to the table size, the index won't be used for equality queries. It might still be used for range queries or as part of a composite index.

## JDBC Batching Configuration

The PostgreSQL JDBC driver has configuration options that significantly affect performance.

### reWriteBatchedInserts

This rewrites individual INSERT statements into multi-row INSERTs:

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb?reWriteBatchedInserts=true
```

Without it, Hibernate's JDBC batching sends:

```sql
INSERT INTO orders (id, status) VALUES (?, ?);
INSERT INTO orders (id, status) VALUES (?, ?);
INSERT INTO orders (id, status) VALUES (?, ?);
```

With it:

```sql
INSERT INTO orders (id, status) VALUES (?, ?), (?, ?), (?, ?);
```

The multi-row INSERT is significantly faster because it's one round trip to the database instead of N. Combined with Hibernate's `batch_size`, this is the single biggest batch performance improvement you can make.

### prepareThreshold

Controls when the driver switches from simple to extended query protocol (server-side prepared statements):

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb?prepareThreshold=5
```

After a query is executed 5 times, the driver sends it as a prepared statement. The database parses and plans it once, then reuses the plan. For frequently executed queries (like JPA's `findById`), this eliminates parsing overhead.

Set it to 0 to disable server-side prepared statements (useful for connection poolers like PgBouncer in transaction mode, which don't support them).

## Statement Caching

The JDBC driver caches prepared statements client-side:

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb?preparedStatementCacheQueries=256&preparedStatementCacheSizeMiB=5
```

This avoids re-creating `PreparedStatement` objects for frequently used queries. The cache is per-connection, so with a pool of 10 connections, you get 10 caches.

## Triggers and Audit Logging

PostgreSQL triggers are the most reliable way to implement audit logging because they fire regardless of how the data was changed - application code, manual SQL, batch jobs, stored procedures.

```sql
CREATE TABLE audit_log (
    id BIGSERIAL PRIMARY KEY,
    table_name TEXT NOT NULL,
    operation TEXT NOT NULL,
    old_data JSONB,
    new_data JSONB,
    changed_at TIMESTAMP DEFAULT NOW(),
    changed_by TEXT
);

CREATE OR REPLACE FUNCTION audit_trigger_func()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO audit_log (table_name, operation, old_data, new_data, changed_by)
    VALUES (
        TG_TABLE_NAME,
        TG_OP,
        CASE WHEN TG_OP IN ('UPDATE', 'DELETE') THEN to_jsonb(OLD) END,
        CASE WHEN TG_OP IN ('INSERT', 'UPDATE') THEN to_jsonb(NEW) END,
        current_setting('app.current_user', true)
    );
    RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER audit_orders
    AFTER INSERT OR UPDATE OR DELETE ON orders
    FOR EACH ROW EXECUTE FUNCTION audit_trigger_func();
```

Set the current user from your Java application:

```java
entityManager.createNativeQuery("SET LOCAL app.current_user = :user")
    .setParameter("user", SecurityContext.getCurrentUser())
    .executeUpdate();
```

`SET LOCAL` scopes the setting to the current transaction, so it auto-resets on commit/rollback. No cleanup needed.

The trigger approach is more reliable than application-level audit logging (Hibernate Envers, event listeners) because it captures all changes, not just those made through JPA. The trade-off is that the audit logic lives in the database, which some teams dislike for maintainability reasons. I think the reliability benefit outweighs the maintenance cost.

## The Takeaway

PostgreSQL is not a dumb storage layer. It's a sophisticated system with features specifically designed for the problems Java developers face - performance, replication, data integrity, and auditing. Learning its internals doesn't mean abandoning JPA. It means using JPA more effectively because you understand what it's doing underneath.

Read the PostgreSQL documentation. It's some of the best technical documentation in existence. And the next time your ORM-generated query is slow, look at `EXPLAIN ANALYZE` before blaming Hibernate. Nine times out of ten, it's a missing index or a bad query plan, not the ORM.
