+++
title = "Apache Flink: When Kafka Streams Isn't Enough"
date = 2026-02-11
[taxonomies]
tags = ["flink", "streaming", "kafka"]
+++

I spent a long time avoiding Flink. Kafka Streams was embedded in our services, it was "just a library," and it handled everything we threw at it. Then we got a requirement for real-time joins across three CDC streams with exactly-once delivery to a Postgres sink, and Kafka Streams started creaking under the weight.

Flink is a different beast. It's a full stream processing platform with its own cluster, its own state management, and its own operational concerns. But for the problems it solves, nothing else comes close.

## The Table/Stream Duality

Flink's conceptual model is built around a powerful idea: a stream and a table are two representations of the same thing. A stream is a sequence of changes. A table is the result of applying those changes. You can convert between them freely.

This isn't just philosophy - it's the core API. Flink lets you treat a Kafka topic as a table (the latest value per key) or as a stream (every event in sequence), and switch between representations as your processing requires.

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
StreamTableEnvironment tableEnv = StreamTableEnvironment.create(env);

// Kafka topic as a dynamic table
tableEnv.executeSql("""
    CREATE TABLE orders (
        order_id STRING,
        customer_id STRING,
        amount DECIMAL(10, 2),
        order_time TIMESTAMP(3),
        WATERMARK FOR order_time AS order_time - INTERVAL '5' SECOND
    ) WITH (
        'connector' = 'kafka',
        'topic' = 'orders',
        'properties.bootstrap.servers' = 'kafka:9092',
        'format' = 'json',
        'scan.startup.mode' = 'earliest-offset'
    )
""");
```

That `WATERMARK` clause is Flink handling event-time processing. It tells Flink that events might arrive up to 5 seconds late, and to hold window results until the watermark has advanced past the window boundary. This is the kind of thing Kafka Streams can do but Flink makes ergonomic.

## Flink SQL with Debezium/CDC Events

This is where Flink became indispensable for us. Debezium CDC streams carry the full change event: `before`, `after`, `op` (operation type), and metadata. Flink has native support for Debezium's JSON format, meaning it can interpret insert, update, and delete operations and maintain a materialized table view.

```sql
CREATE TABLE customers (
    customer_id STRING,
    name STRING,
    email STRING,
    tier STRING,
    PRIMARY KEY (customer_id) NOT ENFORCED
) WITH (
    'connector' = 'kafka',
    'topic' = 'cdc.public.customers',
    'properties.bootstrap.servers' = 'kafka:9092',
    'format' = 'debezium-json',
    'scan.startup.mode' = 'earliest-offset'
);
```

Flink reads the CDC stream, applies the changes, and maintains a live table of current customer state. You can now join this table with other streams in SQL:

```sql
SELECT
    o.order_id,
    o.amount,
    c.name AS customer_name,
    c.tier AS customer_tier
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE c.tier = 'premium';
```

This is a continuously running query. As new orders arrive and customer data changes, the results update in real time. The join is maintained in Flink's state, and the output can be written to another Kafka topic or directly to a database.

For our use case - enriching order events with the latest customer data from a CDC stream - this replaced a custom Kafka Streams application that was managing its own state store and handling CDC deserialization manually. The Flink SQL version is about 20 lines. The Kafka Streams version was 400.

## Running Flink on Kubernetes

Flink on Kubernetes is mature. The Flink Kubernetes Operator manages the lifecycle of Flink clusters and jobs through CRDs.

```yaml
apiVersion: flink.apache.org/v1beta1
kind: FlinkDeployment
metadata:
  name: flink-session-cluster
spec:
  image: flink:1.19-java17
  flinkVersion: v1_19
  flinkConfiguration:
    state.backend: rocksdb
    state.checkpoints.dir: s3://flink-state/checkpoints
    state.savepoints.dir: s3://flink-state/savepoints
    execution.checkpointing.interval: 60s
    taskmanager.memory.process.size: 4096m
    taskmanager.numberOfTaskSlots: "4"
  jobManager:
    resource:
      memory: "2048m"
      cpu: 1
  taskManager:
    replicas: 3
    resource:
      memory: "4096m"
      cpu: 2
```

Two deployment modes:

- **Session mode:** A long-running Flink cluster that accepts multiple jobs. Simpler to manage, but jobs share resources. Good for development and small workloads.
- **Application mode:** One Flink cluster per job. Better resource isolation and scaling. This is what you want in production for important jobs.

### Checkpointing and State

Flink's checkpointing is its secret weapon for exactly-once processing. Periodically, Flink snapshots the state of every operator and the offsets of every source. If a task fails, it restores from the last checkpoint and replays from the saved offsets.

The checkpoint storage matters. Local filesystem is fine for development. For production, use S3 or Azure Blob Storage so checkpoints survive pod restarts and reschedules.

```yaml
state.backend: rocksdb
state.checkpoints.dir: s3://flink-state/checkpoints
execution.checkpointing.interval: 60s
execution.checkpointing.mode: EXACTLY_ONCE
```

RocksDB as the state backend lets Flink manage state larger than available memory by spilling to disk. For jobs with gigabytes of state (think: large windowed aggregations or many-key joins), this is essential.

## When Flink Shines

### Complex Event Processing

Pattern detection across event streams. "Alert when a user logs in from two different countries within an hour." Flink's CEP library handles this natively:

```java
Pattern<LoginEvent, ?> pattern = Pattern.<LoginEvent>begin("first")
    .where(new SimpleCondition<>() {
        public boolean filter(LoginEvent event) { return true; }
    })
    .followedBy("second")
    .where(new IterativeCondition<>() {
        public boolean filter(LoginEvent second, Context<LoginEvent> ctx) {
            LoginEvent first = ctx.getEventsForPattern("first").iterator().next();
            return !second.getCountry().equals(first.getCountry());
        }
    })
    .within(Duration.ofHours(1));
```

Try doing this in Kafka Streams. You can, but you'll be maintaining custom state stores and timer-based eviction logic that's far more error-prone.

### Multi-Source Joins

Flink can read from Kafka, JDBC, files, and other sources in the same job. Joining a Kafka stream with a slowly-changing database dimension table doesn't require putting everything into Kafka first.

### Savepoints for Schema Evolution

Flink's savepoint mechanism lets you stop a job, modify the code (including schema changes), and restart from the saved state. The state is versioned and compatible across code changes, within certain rules. This is incredibly useful for deploying updates to long-running streaming jobs without losing accumulated state.

## The Cost

Flink is not a library you add to your Spring Boot service. It's a platform. You need to run and operate Flink clusters (or use a managed service like AWS Kinesis Data Analytics / Confluent's managed Flink). You need people who understand Flink's execution model, state management, and checkpointing. The learning curve is real.

For simple consume-transform-produce pipelines, Flink is overkill. Kafka Streams or even Spring Cloud Stream will do the job with far less operational overhead.

But for complex joins, CEP, multi-source processing, or jobs with massive state, Flink is the answer. Know where the boundary is for your team, and pick accordingly. We use Kafka Streams for 80% of our stream processing and Flink for the 20% that Kafka Streams can't handle elegantly. That split works well.
