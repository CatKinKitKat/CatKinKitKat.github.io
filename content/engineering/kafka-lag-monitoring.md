+++
title = "Kafka Lag Monitoring: The Metric That Lies to You"
date = 2026-01-14
[taxonomies]
tags = ["kafka", "observability"]
+++

Consumer lag is the most important Kafka metric, and also the most misunderstood. Everyone monitors it. Very few people understand what it actually measures, when it lies, and why the number on your Grafana dashboard might be telling you a story that has nothing to do with reality.

I've spent more hours debugging phantom lag spikes than I care to admit. Here's what I've learned.

## What Lag Actually Is

Consumer lag is the difference between the latest offset in a partition (the log-end offset) and the last committed offset for a consumer group. If the log-end offset is 10,000 and your consumer group has committed up to 9,500, your lag is 500 messages.

Simple enough. The number goes up when producers are faster than consumers, and goes down when consumers catch up. If it keeps growing, your consumers can't keep up with the ingest rate. If it's zero, you're caught up.

Except when you're not.

## The Auto-Commit Lie

If your consumers use auto-commit (the Spring Kafka default), the committed offset might be ahead of what you've actually *processed*. Auto-commit fires on a timer, and it commits the latest *polled* offset, not the latest *processed* offset.

So your lag dashboard says 0, everything looks green, but your consumer is actually 200 messages behind in processing because it polled them but hasn't finished working through the batch. If the consumer crashes, those 200 messages will be reprocessed. Your lag metric completely missed this.

This is why manual offset commit matters, and it's the first thing I check when someone tells me "lag is fine but we're missing messages."

## Compacted Topics: Where Lag Is Meaningless

Log compaction is Kafka's mechanism for keeping only the latest value for each key. Old entries with the same key get removed during compaction. But the offsets don't get reassigned. If you have offsets 1 through 10,000 but compaction has removed 8,000 of them, there are only 2,000 actual messages on the partition.

Your lag metric still says 10,000. It's counting offset positions, not actual messages. You look at the dashboard, panic about being 10,000 messages behind, and start scaling consumers. In reality, you're 2,000 messages behind (maybe less, since many are tombstones your consumer will skip instantly).

I've seen teams double their consumer fleet based on lag numbers from compacted topics. Don't be that team.

## Retention: The Lag That Isn't

Here's another fun one. Your consumer has been offline for a week. The topic has a 3-day retention policy. When the consumer comes back, it can't read from its last committed offset because those segments have been deleted.

What happens depends on `auto.offset.reset`:
- `earliest`: Consumer starts from the beginning of the available log. Lag suddenly shows whatever's in the topic.
- `latest`: Consumer jumps to the end. Lag shows 0. But you just silently skipped a bunch of messages.

Either way, the lag metric doesn't tell you about the messages that were *lost* to retention while the consumer was offline. Those messages are gone. The lag metric can't count what no longer exists.

## The Commit Timestamp Problem

Some monitoring tools (Burrow, for example) use commit timestamps to detect stalled consumers. The idea is: if the committed offset hasn't changed but the timestamp keeps updating, the consumer is alive but not making progress.

Good in theory. The problem: not all consumers commit offsets regularly when there's nothing to process. If a topic has bursty traffic with long idle periods, the commit timestamp goes stale during idle periods. Your monitoring fires an alert saying the consumer is stalled. It isn't - there's just nothing to consume.

We ended up adding a custom health check that distinguishes between "no new messages" and "consumer is stuck." The consumer publishes a heartbeat metric to Prometheus on every poll cycle, regardless of whether it found messages. If the heartbeat stops, *that's* when we know the consumer is in trouble.

## Measuring What Matters

Instead of relying solely on offset-based lag, here's what we actually monitor:

### Records Lag (with context)

Still useful, but interpret it against the topic's characteristics. For a compacted topic, compare lag to the *actual* message count (available via `kafka-log-dirs.sh` or JMX metrics on the broker). For a regular topic, offset lag is reliable.

### Processing Latency

How long does it take from when a message is produced to when it's fully processed? This is the metric your business actually cares about. Measure it by embedding a timestamp in the message payload and comparing it to the processing completion time.

```java
@KafkaListener(topics = "orders")
public void processOrder(ConsumerRecord<String, Order> record) {
    long producedAt = record.timestamp();
    long latency = System.currentTimeMillis() - producedAt;
    meterRegistry.timer("kafka.processing.latency").record(latency, TimeUnit.MILLISECONDS);

    orderService.process(record.value());
}
```

### Consumer Poll Rate

If your consumer's `poll()` loop is taking longer than `max.poll.interval.ms`, it's about to get kicked from the group. Monitor the time between polls. If it's creeping up, you have a processing bottleneck.

### Partition-Level Lag

Aggregate lag across all partitions hides hot-partition problems. If one partition has lag of 50,000 and the other eleven have lag of 0, your aggregate lag looks fine. Monitor per-partition.

## Tooling

- **Burrow** (LinkedIn's open-source tool): Good baseline, evaluates consumer group status with some intelligence about staleness.
- **Prometheus + kafka_exporter**: What we use. Exports lag metrics per partition per consumer group. Combine with custom dashboards.
- **Confluent Control Center**: If you're on Confluent Platform, it's built in and handles most of these nuances.

The tool matters less than understanding what the numbers mean. A fancy dashboard showing bad metrics is worse than a simple one showing the right metrics.

## The Bottom Line

Consumer lag is a useful indicator, not a source of truth. It tells you about offset distance, not processing state. It lies when topics are compacted. It hides silent data loss from retention. And it can show zero when your consumer is actually behind.

Monitor lag, but don't trust it blindly. Combine it with processing latency, poll rate, and partition-level granularity. And for the love of all things operational, switch to manual offset commits so that your lag metric at least reflects what's actually been processed.
