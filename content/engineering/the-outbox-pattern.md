+++
title = "The Outbox Pattern, or How I Stopped Worrying About Dual Writes"
date = 2026-02-15
[taxonomies]
tags = ["kafka", "microservices", "database", "patterns"]
+++

Here's the problem: your service needs to update the database AND publish a message to Kafka. These are two different systems, and you can't do both atomically. If the database write succeeds but the Kafka publish fails (or vice versa), your systems are inconsistent.

This is the dual write problem, and it will absolutely bite you in production if you don't address it. I know because it bit us.

## The Naive Approach

```java
@Transactional
public void createOrder(Order order) {
    orderRepository.save(order);          // 1. write to DB
    kafkaTemplate.send("orders", order);  // 2. publish to Kafka
}
```

If Kafka is down, the DB write succeeds but the message never goes out. If the application crashes between steps 1 and 2, same result. If you swap the order, you get a message about an order that doesn't exist in the database. There's no winning arrangement here.

## The Outbox Pattern

Instead of writing to the database and Kafka separately, you write to the database only - but you include an "outbox" table as part of the same transaction.

```sql
CREATE TABLE outbox (
    id UUID PRIMARY KEY,
    aggregate_type VARCHAR(255),
    aggregate_id VARCHAR(255),
    event_type VARCHAR(255),
    payload JSONB,
    created_at TIMESTAMP DEFAULT NOW()
);
```

Your service writes both the business data and the outbox entry in a single transaction:

```java
@Transactional
public void createOrder(Order order) {
    orderRepository.save(order);
    outboxRepository.save(new OutboxEvent(
        "Order", order.getId(), "OrderCreated", serialize(order)
    ));
}
```

Both writes succeed or both fail. The database guarantees that. Now you need something to pick up those outbox entries and publish them to Kafka.

## Option 1: Polling Publisher

The simplest approach. A scheduled job reads unpublished outbox entries and sends them to Kafka.

```java
@Scheduled(fixedDelay = 1000)
public void publishOutboxEvents() {
    List<OutboxEvent> events = outboxRepository.findUnpublished();
    for (OutboxEvent event : events) {
        kafkaTemplate.send(event.getAggregateType(), event.getPayload());
        event.markPublished();
        outboxRepository.save(event);
    }
}
```

It works. It's simple. The downside is latency - you're polling, so there's always a delay. And you need to handle the case where the Kafka publish succeeds but marking the event as published fails (hello, at-least-once delivery again - make your consumers idempotent).

## Option 2: CDC with Debezium

This is the "proper" solution. Debezium reads the database transaction log (WAL for Postgres, binlog for MySQL) and streams changes to Kafka in near-real-time. When you insert into the outbox table, Debezium picks it up automatically.

The advantage: no polling, low latency, and the order of events matches the transaction order. The disadvantage: you're now running Debezium, Kafka Connect, and managing connectors. It's infrastructure you have to operate.

For our team, the polling approach was the right call initially. We switched to Debezium when latency requirements tightened. Start simple, upgrade when you need to.

## The Inbox Pattern (The Other Side)

The outbox handles the producer side. On the consumer side, you need the inbox pattern to ensure idempotent processing.

```sql
CREATE TABLE inbox (
    event_id UUID PRIMARY KEY,
    processed_at TIMESTAMP DEFAULT NOW()
);
```

Before processing a message, check if you've already processed it:

```java
@KafkaListener(topics = "orders")
public void handleOrderEvent(OrderEvent event) {
    if (inboxRepository.existsById(event.getId())) {
        return; // already processed, skip
    }

    // process the event
    inventoryService.reserveStock(event);

    // mark as processed (same transaction as the business logic)
    inboxRepository.save(new InboxEntry(event.getId()));
}
```

Same idea as the outbox but in reverse. The business processing and the inbox insert happen in the same transaction, so you can't process a message twice.

## When You Don't Need This

If your system can tolerate occasional inconsistency and you have manual reconciliation processes... honestly, you might not need the outbox pattern. It adds complexity. A simple retry mechanism on the Kafka publish might be enough for non-critical flows.

But for anything where data consistency matters - orders, payments, inventory - the outbox pattern is the correct solution. It's boring, it's extra infrastructure, and it works.
