+++
title = "Kafka Consumer Offsets: What Nobody Tells You"
date = 2026-01-25
[taxonomies]
tags = ["kafka", "spring-boot", "messaging"]
+++

I've been working with Kafka long enough to know that the documentation gives you the theory, but production gives you the education. Consumer offsets are one of those things that seem simple until they aren't.

## The Basics (Quick, I Promise)

Kafka tracks where each consumer group has read up to in each partition via offsets. When your consumer processes a message and commits the offset, Kafka knows not to send it again. Simple.

Except it's not simple. Because the question isn't "how do offsets work" - it's "when exactly should I commit, and what happens when I get it wrong?"

## Auto-Commit: The Default That Will Burn You

Spring Boot's Kafka consumer defaults to auto-commit with a 5-second interval (`enable.auto.commit=true`, `auto.commit.interval.ms=5000`). This means offsets get committed on a timer, completely disconnected from whether you've actually finished processing the message.

The scenario that will ruin your day:

1. Consumer picks up message at offset 42
2. Starts processing - calls an external API, writes to the database
3. Auto-commit fires, commits offset 42
4. External API call fails
5. Your application throws an exception
6. Kafka thinks offset 42 is done. The message is gone.

You just lost data. Not because Kafka failed - because your commit strategy was wrong.

## Manual Commit: The Right Way (With Caveats)

Disable auto-commit. Commit after you've successfully processed the message.

```java
@KafkaListener(topics = "orders")
public void processOrder(ConsumerRecord<String, Order> record, Acknowledgment ack) {
    try {
        orderService.process(record.value());
        ack.acknowledge(); // commit only after success
    } catch (Exception e) {
        // don't acknowledge - message will be redelivered
        log.error("Failed to process order at offset {}", record.offset(), e);
    }
}
```

Spring Kafka's `AckMode.MANUAL_IMMEDIATE` is what you want here. Set it in your container factory:

```java
factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL_IMMEDIATE);
```

## The At-Least-Once Reality

Manual commit gives you at-least-once delivery. Your consumer might process the same message twice if it crashes after processing but before committing. This means your processing logic needs to be idempotent.

For us, that means either:
- Using a database unique constraint on a business key derived from the message
- Keeping a processed-message-id table and checking before processing
- Making the operation naturally idempotent (e.g., "set status to X" rather than "increment counter")

The third option is the best when you can get away with it. The first is the most practical. The second is the most explicit but adds latency.

## Batch vs Record Acknowledgment

If you're processing messages one at a time, `MANUAL_IMMEDIATE` works. But if you're using batch listeners for throughput, you need to think about what happens when message 3 out of 10 fails.

If you commit the batch offset, you've committed messages 4-10 as processed even though you didn't process them. If you don't commit, you'll reprocess messages 1 and 2 that already succeeded.

The pattern that's worked for me: commit per-record within the batch loop, and on failure, seek back to the failed offset.

```java
@KafkaListener(topics = "events", batch = "true")
public void processBatch(List<ConsumerRecord<String, Event>> records,
                         Consumer<String, Event> consumer) {
    for (ConsumerRecord<String, Event> record : records) {
        try {
            eventService.process(record.value());
        } catch (Exception e) {
            // seek back to this offset, everything from here will be retried
            consumer.seek(
                new TopicPartition(record.topic(), record.partition()),
                record.offset()
            );
            return;
        }
    }
}
```

## Rebalancing: The Offset Killer

Consumer group rebalancing is where offset problems get creative. When partitions get reassigned between consumers, there's a window where the new consumer might re-read messages the old consumer already processed (but hadn't committed yet).

Spring Kafka's `ConsumerRebalanceListener` lets you commit offsets before partitions get revoked:

```java
factory.getContainerProperties().setConsumerRebalanceListener(new ConsumerAwareRebalanceListener() {
    @Override
    public void onPartitionsRevokedBeforeCommit(Consumer<?, ?> consumer,
                                                 Collection<TopicPartition> partitions) {
        consumer.commitSync(); // flush pending offsets before losing partitions
    }
});
```

This doesn't eliminate rebalance-related duplicates entirely, but it shrinks the window significantly.

## The Real Lesson

Offsets aren't a Kafka feature you configure once and forget. They're a contract between your application and the broker about what "processed" means. Get that contract wrong, and you either lose messages or process them forever.

The boring answer is: disable auto-commit, acknowledge manually, make your processing idempotent, and handle rebalancing. It's not glamorous. But the alternative is debugging data loss at 2 AM, and I've done enough of that.
