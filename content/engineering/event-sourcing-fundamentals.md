+++
title = "Event Sourcing Fundamentals: Storing What Happened, Not Where You Are"
date = 2026-02-26
[taxonomies]
tags = ["event-sourcing", "architecture", "patterns"]
+++

I'll admit it: the first time someone explained event sourcing to me, I thought it was overengineered nonsense. "Instead of storing the current state, store every event that led to the current state." That sounded like storing your entire browser history instead of a bookmark. Why would you do that?

Then I worked on a system where we needed to reconstruct how an account balance reached a negative value three weeks ago, and the only answer was "we don't know, we only have the current state." That's when event sourcing stopped being academic and started being practical.

## The Concept

In traditional systems, you store the current state of an entity. An order has a status of "SHIPPED." If you want to know when it was placed, confirmed, packed, and shipped, you need separate audit logging, which is often an afterthought, incomplete, or wrong.

Event sourcing flips this. You store the sequence of events that happened:

```
OrderPlaced      { orderId: 42, items: [...], timestamp: T1 }
PaymentReceived  { orderId: 42, amount: 99.50, timestamp: T2 }
OrderPacked      { orderId: 42, warehouseId: 7, timestamp: T3 }
OrderShipped     { orderId: 42, trackingNumber: "XYZ", timestamp: T4 }
```

The current state (order is shipped) is derived by replaying these events. You always know exactly how you got here, because the history *is* the data.

## Implementing with a Relational Database

You don't need a fancy event store to start with event sourcing. A PostgreSQL table works fine:

```sql
CREATE TABLE events (
    id BIGSERIAL PRIMARY KEY,
    aggregate_type VARCHAR(100) NOT NULL,
    aggregate_id VARCHAR(100) NOT NULL,
    event_type VARCHAR(100) NOT NULL,
    version INT NOT NULL,
    payload JSONB NOT NULL,
    metadata JSONB,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (aggregate_type, aggregate_id, version)
);

CREATE INDEX idx_events_aggregate ON events (aggregate_type, aggregate_id, version);
```

The `version` column with the unique constraint is the key to optimistic concurrency. When you append a new event, you specify the expected version. If another write snuck in first, the unique constraint violation tells you there's a conflict.

```java
@Repository
public class EventStore {

    private final JdbcTemplate jdbc;

    public void append(String aggregateType, String aggregateId,
                       int expectedVersion, List<DomainEvent> events) {
        int version = expectedVersion;
        for (DomainEvent event : events) {
            version++;
            jdbc.update(
                "INSERT INTO events (aggregate_type, aggregate_id, event_type, version, payload) " +
                "VALUES (?, ?, ?, ?, ?::jsonb)",
                aggregateType, aggregateId, event.getClass().getSimpleName(),
                version, serialize(event)
            );
        }
    }

    public List<DomainEvent> load(String aggregateType, String aggregateId) {
        return jdbc.query(
            "SELECT * FROM events WHERE aggregate_type = ? AND aggregate_id = ? ORDER BY version",
            new Object[]{aggregateType, aggregateId},
            this::mapEvent
        );
    }
}
```

If the insert fails with a unique constraint violation on `(aggregate_type, aggregate_id, version)`, you have a concurrent modification. Reload, re-apply your business logic, retry. This is exactly how you'd handle optimistic locking in a traditional system, just with events instead of row versions.

## Event Store Design Decisions

Building a production event store means making choices that aren't obvious from tutorials:

**Serialization format:** JSON (specifically JSONB in PostgreSQL) is my default. It's queryable, human-readable, and schema-flexible. Avro or Protobuf are better for throughput-critical stores, but JSON's debuggability wins for most use cases.

**Event schema evolution:** Events are immutable. You can't change a published event. But your event schemas will evolve. My approach: event upcasters that transform old event versions into the current format during deserialization. It's tedious but it works.

```java
public class OrderPlacedV1ToV2Upcaster implements EventUpcaster {
    public DomainEvent upcast(JsonNode oldEvent) {
        // V1 had "price", V2 has "amount" - rename the field
        ObjectNode node = (ObjectNode) oldEvent;
        node.set("amount", node.remove("price"));
        return deserialize(node, OrderPlacedV2.class);
    }
}
```

**Snapshotting:** For aggregates with long event histories (thousands of events), replaying from the beginning gets slow. Snapshots save a point-in-time state that you can start replaying from:

```sql
CREATE TABLE snapshots (
    aggregate_type VARCHAR(100) NOT NULL,
    aggregate_id VARCHAR(100) NOT NULL,
    version INT NOT NULL,
    state JSONB NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (aggregate_type, aggregate_id)
);
```

Take a snapshot every N events (I use 100 as a starting point), and when loading, start from the snapshot and replay only the events after it. This turned a 2-second aggregate load into a 50ms load on one of our systems.

**Partitioning:** Once your event table hits millions of rows, you'll want to partition. Partitioning by aggregate_type or by time range (monthly partitions) keeps queries fast. PostgreSQL's native partitioning handles this well:

```sql
CREATE TABLE events (
    - same columns as before
) PARTITION BY RANGE (created_at);

CREATE TABLE events_2026_01 PARTITION OF events
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
```

## Lessons Learned the Hard Way

### Modeling: Events Are Not CRUD Operations

The most common mistake I see (and made myself): modeling events as "EntityCreated," "EntityUpdated," "EntityDeleted." That's just CRUD with extra steps. Events should capture *business intent*:

- Bad: `OrderUpdated { status: "SHIPPED" }`
- Good: `OrderShipped { trackingNumber: "XYZ", carrier: "DHL" }`

The difference matters because "OrderShipped" carries semantic meaning that enables downstream consumers to react appropriately, while "OrderUpdated" requires every consumer to parse the payload and figure out what changed.

### Consistency: Embrace the Aggregate Boundary
Events within a single aggregate are strongly consistent; the version-based optimistic concurrency ensures it. Events across aggregates are eventually consistent. Trying to make cross-aggregate operations transactional defeats the purpose of event sourcing and leads to distributed transaction nightmares.

## The Projectionist (Read Side)
If your business operation spans multiple aggregates, use a saga or process manager. Accept that the operation isn't atomic and design for compensating actions when things go wrong.

### Storage: It Grows Fast

Event stores are append-only by design. They grow. A system processing 1,000 events per second generates 86 million events per day. After a year, you're looking at 31 billion events. Plan for this from the start.

Archival strategy matters. Move old events to cold storage (S3, Azure Blob) after a retention period. Keep recent events in the hot store for fast access. Make sure your snapshotting strategy means you rarely need to access ancient events.

## Akka Durable State: An Alternative Approach

Akka (now Apache Pekko) offers an interesting middle ground with its durable state pattern. Instead of storing events, you store the current state of the actor, but the actor's behavior is still event-driven internally.

```java
// Akka-style (conceptual - actual Akka syntax is more involved)
public class OrderActor extends EventSourcedBehavior<OrderCommand, OrderEvent, OrderState> {

    @Override
    public OrderState applyEvent(OrderEvent event, OrderState state) {
        return switch (event) {
            case OrderPlaced e -> state.withStatus("PLACED").withItems(e.items());
            case OrderShipped e -> state.withStatus("SHIPPED").withTracking(e.trackingNumber());
            default -> state;
        };
    }
}
```

The durable state approach persists snapshots instead of events, which simplifies storage and recovery at the cost of losing the event history. For systems where you care about the current state but not the journey, this is a pragmatic choice.

## PostgreSQL Benchmarks

Because everyone asks: how does a PostgreSQL-based event store perform?

On a system I benchmarked last year (PostgreSQL 16, standard Azure Flexible Server, 4 vCores, 16GB RAM):

- **Write throughput:** ~8,000 events/second sustained, ~15,000 peak
- **Read throughput (single aggregate, 100 events):** ~2ms
- **Read throughput (single aggregate, 10,000 events):** ~180ms (this is why you snapshot)
- **Read throughput (single aggregate, 10,000 events, with snapshot):** ~5ms

For most business applications, this is more than sufficient. If you need hundreds of thousands of events per second, you're looking at EventStoreDB, Kafka as an event store, or a custom solution. But Postgres gets you surprisingly far, and your team probably already knows how to operate it.

## When to Use Event Sourcing

Event sourcing is not a default architecture. It adds complexity in exchange for capabilities. Use it when:

- You need a complete audit trail (finance, healthcare, compliance)
- You need to reconstruct past states ("what did this look like on March 3rd?")
- Your domain is naturally event-driven (order processing, workflow management)
- You need event replay for debugging or reprocessing

Don't use it when:
- Simple CRUD is sufficient (most admin interfaces, user management)
- You don't need history and won't need history
- Your team doesn't have the bandwidth to learn and maintain it

The cost of event sourcing is not in the initial implementation. It's in the ongoing operational complexity: event schema evolution, storage growth, projection management, and the conceptual overhead of a model that's fundamentally different from what most developers are used to. Make sure the benefits justify that cost.
