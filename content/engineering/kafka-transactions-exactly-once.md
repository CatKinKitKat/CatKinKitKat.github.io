+++
title = "Kafka Transactions and the Myth of Exactly-Once"
date = 2026-01-05
[taxonomies]
tags = ["kafka", "spring-boot", "messaging"]
+++

"Exactly-once semantics" is one of those phrases that gets thrown around in Kafka conversations like it's a checkbox you tick and move on. It isn't. It's a set of trade-offs that most teams don't fully understand until they're debugging duplicate records in production at 11 PM on a Thursday.

Let me walk you through what Kafka actually gives you, what it costs, and where the whole thing falls apart.

## Idempotent Producers: The Foundation

Before we talk about transactions, we need to talk about idempotent producers. This is Kafka's answer to the question: "What happens when the broker acks a write, but the ack gets lost, and the producer retries?"

Without idempotency, you get duplicate messages. The producer doesn't know the first write succeeded, so it sends it again. The broker happily accepts both. Your downstream consumers now process the same event twice.

Setting `enable.idempotence=true` (which is the default since Kafka 3.0) assigns each producer a PID and sequence number. The broker uses these to deduplicate retries. Same PID, same sequence number, same partition - the broker knows it's a retry and silently drops it.

This is free. There's almost no performance penalty. If you're not using it, you should be. But idempotent producers only protect you against *producer retries*. They don't help with the read-process-write pattern that most stream processing applications use.

## Transactional Producers: The Real Deal

Transactions are where Kafka starts earning the "exactly-once" label. A transactional producer can atomically write to multiple partitions (and multiple topics) and commit consumer offsets, all as a single unit.

```java
@Bean
public ProducerFactory<String, String> producerFactory() {
    Map<String, Object> config = new HashMap<>();
    config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
    config.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "order-processor-tx");
    config.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
    return new DefaultKafkaProducerFactory<>(config);
}
```

The `transactional.id` is the key. It identifies this producer across restarts, allowing the broker to fence off zombie instances. If an old producer with the same transactional ID tries to write after a new one has started, its writes get rejected. This is how Kafka prevents duplicate processing after a crash and restart.

In Spring Kafka, the transaction flow looks like this:

```java
kafkaTemplate.executeInTransaction(ops -> {
    ops.send("output-topic", key, transformedValue);
    ops.sendOffsetsToTransaction(
        Map.of(new TopicPartition("input-topic", partition),
               new OffsetAndMetadata(offset + 1)),
        consumerGroupId
    );
    return true;
});
```

The consumer offset commit and the output write either both happen or neither does. That's the atomicity guarantee.

## Consumer Side: read_committed

None of this matters if your consumers aren't configured to respect transactions. By default, consumers read *all* messages, including those from aborted transactions. You need to set `isolation.level=read_committed`.

```properties
spring.kafka.consumer.properties.isolation.level=read_committed
```

With `read_committed`, consumers only see messages from committed transactions. They also won't read past the "last stable offset" (LSO) - the offset of the earliest open transaction. This means a slow or stuck transaction will block *all* consumers on that partition from making progress. Fun.

## The Practical Limits

Here's where the "exactly-once" narrative starts to crack.

Kafka's transactional guarantees are scoped to *Kafka itself*. The moment your processing involves an external system - a database write, an HTTP call, a cache update - you're outside the transaction boundary. If your consumer reads from Kafka, writes to Postgres, and then the offset commit fails, you'll process that message again. Your Postgres write needs to be idempotent.

This isn't a Kafka bug. It's a fundamental limitation of distributed systems. True exactly-once across system boundaries requires two-phase commit, and nobody wants that (for good reason).

The other limit is performance. Transactions add latency. The broker has to coordinate across partitions, maintain transaction state, and handle the two-phase commit protocol internally. For high-throughput systems, this can be significant. We measured roughly 20-30% throughput reduction when enabling transactions in one of our pipelines. Whether that's acceptable depends on your use case.

## The Zombie Fencing Problem

I mentioned `transactional.id` earlier. Here's the scenario it solves: your consumer-processor crashes after reading a batch of messages and writing output, but before committing offsets. Kafka's consumer group rebalances, and another instance picks up the same partitions. It reads the same messages and processes them again.

Without transactions, you get duplicates. With transactions and proper `transactional.id` configuration, the new instance fences off the old one. Any in-flight transactions from the crashed instance get aborted. Clean.

But the `transactional.id` has to be deterministic and tied to the input partition. If you use random IDs, you lose zombie fencing. If you use a single ID for all instances, you get contention. The typical pattern is `{application-id}-{partition}`.

## When to Actually Use This

I'll be honest: most of our services don't use Kafka transactions. The overhead - both in performance and in operational complexity - isn't justified when you can achieve the same practical result with idempotent consumers and the outbox pattern.

We use transactions in exactly two scenarios:
1. Stream processing pipelines that read from Kafka and write back to Kafka (the consume-transform-produce pattern)
2. Cases where we absolutely cannot tolerate any duplicates and the downstream system has no natural idempotency mechanism

For everything else, at-least-once with idempotent consumers is simpler, faster, and easier to debug. The boring answer, as usual, is the right one.

## The Bottom Line

Kafka's "exactly-once" is real, but it's narrower than the marketing suggests. It works within Kafka's boundaries. It requires careful configuration. It costs throughput. And it doesn't save you from making your external integrations idempotent.

Understand what it gives you, understand what it doesn't, and then make an informed decision. That's more than most teams do.
