+++
title = "CDC with Debezium - Because Your Database Is Already an Event Stream"
date = 2025-09-15
[taxonomies]
tags = ["database", "kafka", "cdc"]
+++

Here's a take that took me too long to internalize: your database transaction log is already an ordered, durable stream of every data change in your system. Change Data Capture (CDC) is just the practice of reading that stream and doing something useful with it. Debezium is the tool that makes it practical.

## What CDC Actually Is

Every write to a relational database - INSERT, UPDATE, DELETE - is recorded in the transaction log before it's applied to the tables. PostgreSQL calls it the WAL (Write-Ahead Log). MySQL calls it the binlog. Oracle calls it the redo log. These logs exist for crash recovery and replication, but they're also a complete, ordered history of every data change.

CDC taps into this log and streams the changes to an external system (usually Kafka). Your application doesn't change. It keeps writing to the database normally. Debezium reads the log and publishes events.

The key insight: CDC captures changes at the database level, not the application level. If someone runs a manual SQL UPDATE, CDC captures it. If a stored procedure modifies data, CDC captures it. If a different microservice writes to a shared table, CDC captures it. It's comprehensive in a way that application-level event publishing can never be.

## Debezium on PostgreSQL

PostgreSQL CDC requires logical replication, which decodes the WAL into a structured stream of changes.

### Setup

1. Enable logical replication in `postgresql.conf`:

```
wal_level = logical
max_replication_slots = 4
max_wal_senders = 4
```

2. Create a publication (PostgreSQL 10+):

```sql
CREATE PUBLICATION my_publication FOR TABLE orders, customers, line_items;
```

3. Configure the Debezium connector (via Kafka Connect):

```json
{
    "name": "postgres-connector",
    "config": {
        "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
        "database.hostname": "postgres",
        "database.port": "5432",
        "database.user": "debezium",
        "database.password": "secret",
        "database.dbname": "mydb",
        "topic.prefix": "myapp",
        "table.include.list": "public.orders,public.customers",
        "plugin.name": "pgoutput",
        "slot.name": "debezium_slot",
        "publication.name": "my_publication"
    }
}
```

Debezium creates a replication slot and starts streaming changes. Each table gets a Kafka topic: `myapp.public.orders`, `myapp.public.customers`, etc.

### The Replication Slot Warning

Replication slots track how far a consumer has read in the WAL. If Debezium goes down and the slot isn't consumed, PostgreSQL retains WAL segments. Disk usage grows. If it grows enough, your database runs out of disk. I've seen this happen in production.

Monitor `pg_replication_slots`:

```sql
SELECT slot_name, active, pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) AS lag_bytes
FROM pg_replication_slots;
```

Set `max_slot_wal_keep_size` in PostgreSQL 13+ to cap the WAL retention. And monitor the lag - if Debezium falls behind, you need to know before the disk fills up.

## Debezium on MySQL

MySQL CDC reads the binary log (binlog).

```sql
-- Enable binlog in my.cnf
server-id = 1
log_bin = mysql-bin
binlog_format = ROW
binlog_row_image = FULL
```

`binlog_format = ROW` is critical. Statement-based replication logs SQL statements, which Debezium can't reliably decode. Row-based replication logs the actual row changes.

```json
{
    "name": "mysql-connector",
    "config": {
        "connector.class": "io.debezium.connector.mysql.MySqlConnector",
        "database.hostname": "mysql",
        "database.port": "3306",
        "database.user": "debezium",
        "database.server.id": "184054",
        "database.include.list": "mydb",
        "topic.prefix": "myapp",
        "schema.history.internal.kafka.bootstrap.servers": "kafka:9092",
        "schema.history.internal.kafka.topic": "schema-changes.mydb"
    }
}
```

MySQL requires Debezium to track schema history (DDL changes) separately, because the binlog doesn't include the schema at the time of the change. This is one area where PostgreSQL's logical replication is cleaner.

## Taxonomy of Change Events

Debezium events have a consistent structure:

```json
{
    "before": { "id": 1, "status": "PENDING", "total": 100.00 },
    "after":  { "id": 1, "status": "SHIPPED", "total": 100.00 },
    "source": {
        "version": "2.4.0",
        "connector": "postgresql",
        "ts_ms": 1694700000000,
        "txId": 12345,
        "lsn": 123456789
    },
    "op": "u",
    "ts_ms": 1694700000500
}
```

The `op` field tells you what happened:
- `c` - create (INSERT)
- `u` - update (UPDATE)
- `r` - read (snapshot, initial load)
- `d` - delete (DELETE)

For deletes, `after` is null. For creates, `before` is null. For updates, both are present, giving you the before and after state. This is incredibly useful for audit logging, cache invalidation, and detecting specific field changes.

## The Outbox Pattern with Debezium

This is where CDC becomes truly powerful. Instead of publishing events from your application (dual write problem), you write to an outbox table and let Debezium pick up the changes.

```sql
CREATE TABLE outbox (
    id UUID PRIMARY KEY,
    aggregate_type VARCHAR(255),
    aggregate_id VARCHAR(255),
    type VARCHAR(255),
    payload JSONB
);
```

Debezium has built-in support for the outbox pattern via the Outbox Event Router:

```json
{
    "transforms": "outbox",
    "transforms.outbox.type": "io.debezium.transforms.outbox.EventRouter",
    "transforms.outbox.table.fields.additional.placement": "type:header:eventType",
    "transforms.outbox.route.topic.replacement": "events.${routedByValue}"
}
```

The event router reads the outbox table changes, extracts the payload, routes to the appropriate Kafka topic based on the aggregate type, and optionally deletes the outbox row after processing. It turns your outbox table into a proper event bus, with exactly-once semantics (at the database level) and no dual write problem.

## Use Cases Beyond Event Publishing

CDC isn't just for event-driven architecture. Other uses I've seen in production:

**Cache invalidation:** When a row changes in the database, publish a CDC event that invalidates the corresponding cache entry. No more stale caches because someone updated the database directly.

**Search index sync:** Stream database changes to Elasticsearch or OpenSearch. The search index stays in sync with the database without your application needing to write to both.

**Data replication:** Stream changes from the primary database to a data warehouse, analytics system, or read replica in a different technology.

**Audit logging:** Every change event includes before and after values. Write them to an audit log. Complete, reliable, and decoupled from the application.

## "CDC Is a Feature, Not a Product"

This is the perspective shift that matters. CDC isn't something you "add to your architecture." Your database is already capturing every change. CDC is just exposing that capability.

Debezium is the most popular tool for this, but the underlying mechanism - logical replication in PostgreSQL, binlog in MySQL - is a database feature. You could consume the WAL directly if you wanted to. Debezium just handles the hard parts: initial snapshots, schema changes, offset tracking, and the Kafka integration.

The decision isn't "should we use CDC?" The decision is "are we ignoring a data stream that already exists?" Every time I've adopted CDC in a project, I've found uses for it that weren't in the original plan. Audit logs, analytics feeds, cache invalidation - they all became easier because the change stream was already there.

## The Operational Reality

Debezium adds moving parts: Kafka Connect, connectors, replication slots, schema history topics. It needs monitoring, and it needs someone who understands both Kafka and database replication to debug issues.

The most common operational issues:

1. **Replication slot bloat** - monitor and alert on WAL lag
2. **Schema changes** - DDL changes can break the connector if not handled carefully
3. **Connector restart after long downtime** - the replication slot may have expired, requiring a new snapshot
4. **Kafka Connect worker failures** - run multiple workers for high availability

It's infrastructure. It needs care. But the alternative - application-level event publishing with all its consistency problems - needs even more care, and it's less reliable.

Start with a single table, a single connector, and a simple consumer. Prove the value. Then expand. CDC adoption should be incremental, not big-bang.
