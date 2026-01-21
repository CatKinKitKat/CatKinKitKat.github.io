+++
title = "Kafka Streams: The Stream Processing Library That Lives in Your Application"
date = 2026-01-21
[taxonomies]
tags = ["kafka", "streaming", "spring-boot"]
+++

The first time someone told me Kafka Streams was "just a library," I didn't believe them. Stream processing meant clusters. It meant Flink or Spark Streaming. It meant a separate infrastructure team managing a separate platform. The idea that you could do stateful stream processing from inside a regular Spring Boot application felt like a trick.

It's not a trick. It's genuinely just a library. And for a wide range of stream processing use cases, it's the right tool.

## The Programming Model

Kafka Streams operates on two core abstractions: **KStream** (an unbounded stream of events) and **KTable** (a changelog stream that represents the current state of a table). Both are backed by Kafka topics.

A KStream is what you'd expect: every record is an independent event. A KTable is a materialized view where each key has exactly one current value. Insert a record with key "A" and value 1, then another with key "A" and value 2 - the KTable shows key "A" = 2.

```java
StreamsBuilder builder = new StreamsBuilder();

KStream<String, Order> orders = builder.stream("orders");
KTable<String, Long> orderCounts = orders
    .groupByKey()
    .count(Materialized.as("order-counts-store"));

orderCounts.toStream().to("order-counts");
```

This reads from the `orders` topic, groups by key, counts per key, materializes the count in a local state store, and writes the results to `order-counts`. All of this runs inside your application process. No external cluster needed.

## Stateful Operations

The real power of Kafka Streams is stateful operations: aggregations, joins, and windowed computations. These require state, and Kafka Streams manages that state for you using RocksDB-backed local stores.

### Joins

You can join a KStream with a KTable (enrichment) or two KStreams (correlation within a time window).

```java
KStream<String, Order> orders = builder.stream("orders");
KTable<String, Customer> customers = builder.table("customers");

KStream<String, EnrichedOrder> enriched = orders.join(
    customers,
    (order, customer) -> new EnrichedOrder(order, customer)
);
```

This joins every order with the current customer record, using the message key. The KTable is always up to date because it's continuously consuming the `customers` topic. When a customer record changes, the next order for that customer gets the updated information.

### Windowed Aggregations

Time-windowed aggregations are where Kafka Streams shines for real-time analytics.

```java
KStream<String, Transaction> transactions = builder.stream("transactions");

KTable<Windowed<String>, Long> windowedCounts = transactions
    .groupByKey()
    .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofMinutes(5)))
    .count(Materialized.as("txn-counts-5min"));
```

Five-minute tumbling windows, counting transactions per key. The state store handles late arrivals (within the window), and you can configure grace periods for events that arrive slightly out of order.

## SAGA Pattern with Kafka Streams

The SAGA pattern for distributed transactions works naturally with Kafka Streams. Each step in the saga is a stream processing stage that reads events, performs local processing, and emits the next event (or a compensation event on failure).

```java
KStream<String, OrderEvent> orderEvents = builder.stream("order-events");

KStream<String, OrderEvent>[] branches = orderEvents.branch(
    (key, event) -> event.getType().equals("CREATED"),
    (key, event) -> event.getType().equals("PAYMENT_FAILED"),
    (key, event) -> true // default
);

// Happy path: order created -> reserve inventory
branches[0]
    .mapValues(event -> new InventoryReservation(event.getOrderId(), event.getItems()))
    .to("inventory-commands");

// Compensation: payment failed -> release inventory
branches[1]
    .mapValues(event -> new InventoryRelease(event.getOrderId()))
    .to("inventory-commands");
```

Each service in the saga handles its own step and emits success or failure events. The orchestration is implicit in the event flow rather than managed by a central coordinator. Kafka's durability guarantees that events aren't lost between steps, and exactly-once processing (with transactions enabled) prevents duplicate saga executions.

The key advantage over a centralized orchestrator: each service owns its step. There's no single point of failure coordinating the whole flow. The downside: debugging a distributed saga across multiple topics and services requires good tracing and correlation IDs.

## ksqlDB: SQL Over Streams

ksqlDB puts a SQL interface on top of Kafka Streams. Instead of writing Java, you write SQL-like queries that compile down to Kafka Streams topologies.

```sql
CREATE STREAM orders_stream (
  order_id VARCHAR KEY,
  customer_id VARCHAR,
  amount DECIMAL(10,2)
) WITH (KAFKA_TOPIC='orders', VALUE_FORMAT='AVRO');

CREATE TABLE orders_per_customer AS
  SELECT customer_id, COUNT(*) AS order_count, SUM(amount) AS total_amount
  FROM orders_stream
  WINDOW TUMBLING (SIZE 1 HOUR)
  GROUP BY customer_id;
```

For analytics and simple transformations, ksqlDB is faster to develop and easier to maintain than custom Java code. For complex business logic with branching, error handling, and external service calls, stick with the Java API.

One caveat: ksqlDB is a Confluent project, not part of Apache Kafka. If you're running open-source Kafka, you'll need to evaluate whether adding ksqlDB to your stack is worth the dependency.

## When to Use Kafka Streams vs Flink

This is the question I get asked most. Here's my framework:

**Use Kafka Streams when:**
- Your input and output are both Kafka topics
- You want stream processing embedded in your microservices (no separate cluster)
- Your team knows Java/Spring Boot and doesn't want to learn a new platform
- Your processing is "per-application" - each service handles its own stream logic

**Use Flink when:**
- You need to process data from non-Kafka sources (databases, files, APIs)
- You need complex event processing with advanced windowing (session windows, pattern detection)
- You're doing centralized stream processing for analytics or data engineering
- You need to handle massive state (hundreds of GB) with incremental checkpointing
- You need exactly-once processing across non-Kafka sinks

The simplest way to think about it: Kafka Streams is a library for microservices that happen to do stream processing. Flink is a platform for stream processing that happens to integrate with microservices.

## Operational Considerations

Kafka Streams state stores are local to each application instance. They're backed by changelog topics in Kafka, so they survive restarts (the state is rebuilt from the changelog on startup). But rebuilding state takes time. For large state stores, this can mean minutes of initialization.

Standby replicas help: you can configure `num.standby.replicas` to maintain hot copies of state stores on other instances. When a rebalance happens, the standby can take over immediately instead of rebuilding from scratch.

```properties
num.standby.replicas=1
```

The other thing to watch: RocksDB memory usage. Each state store uses RocksDB under the hood, and the default memory allocation can add up fast. Configure RocksDB's block cache size explicitly, or you'll wonder why your 512MB container keeps getting OOM-killed.

Kafka Streams doesn't get the hype that Flink does, but for the right use cases, it's the simpler and more practical choice. It's a library, not a platform. Sometimes that's exactly what you need.
