+++
title = "Kafka Partitioning: Where Your Scaling Strategy Lives or Dies"
date = 2026-01-07
[taxonomies]
tags = ["kafka", "distributed-systems"]
+++

Partitioning is the mechanism that makes Kafka scalable. It's also the mechanism that causes the most confusion when things go sideways. I've seen teams add partitions thinking it would fix a throughput problem, only to discover they'd introduced ordering bugs that took weeks to track down.

Let's talk about what partitioning actually does, the strategies available, and why consumer group rebalancing is the thing that will wake you up at night.

## The Basics

A Kafka topic is split into partitions. Each partition is an ordered, immutable log. Messages within a partition are guaranteed to be in order. Messages across partitions have no ordering guarantee whatsoever. This is the single most important thing to understand about Kafka, and the thing most people get wrong first.

A partition is the unit of parallelism. One partition can be read by exactly one consumer in a consumer group. If you have 12 partitions and 12 consumers, each consumer gets one partition. If you have 12 partitions and 15 consumers, three consumers sit idle. If you have 12 partitions and 4 consumers, each consumer gets three partitions.

This means the maximum parallelism for a single consumer group is capped by the number of partitions. Choose this number carefully, because changing it later has consequences.

## Key-Based Partitioning

The default partitioning strategy uses the message key. Kafka hashes the key (murmur2 by default) and maps it to a partition: `partition = hash(key) % numPartitions`.

This guarantees that all messages with the same key land on the same partition, which means they're processed in order by the same consumer. For order events keyed by `orderId`, this means all events for a given order are processed sequentially. Exactly what you want.

The trap: if your keys aren't evenly distributed, you get hot partitions. One customer generating 80% of your traffic means one partition (and one consumer) doing 80% of the work while the others are idle. We had this exact problem with a B2B system where one large client generated orders at 10x the volume of everyone else. The fix was a composite key that distributed their traffic more evenly.

## Round-Robin: When Order Doesn't Matter

If you send messages without a key, Kafka uses a sticky partitioner (since KIP-480). It batches messages to the same partition for a while, then switches. This maximizes batch efficiency while still distributing messages across partitions.

Before the sticky partitioner, keyless messages used pure round-robin, which produced tiny batches and killed throughput. If you're on an older Kafka client, consider upgrading just for this.

Round-robin is the right choice when you don't care about per-key ordering - think logging, metrics collection, or any fire-and-forget workload where throughput matters more than sequence.

## Custom Partitioners

Sometimes the default hash-based strategy doesn't cut it. You need a custom partitioner.

```java
public class PriorityPartitioner implements Partitioner {
    @Override
    public int partition(String topic, Object key, byte[] keyBytes,
                         Object value, byte[] valueBytes, Cluster cluster) {
        int numPartitions = cluster.partitionsForTopic(topic).size();
        if (key != null && key.toString().startsWith("PRIORITY-")) {
            return 0; // dedicated partition for priority messages
        }
        return Utils.toPositive(Utils.murmur2(keyBytes)) % (numPartitions - 1) + 1;
    }
}
```

We've used custom partitioners for tenant isolation (each major tenant gets dedicated partitions) and for priority routing. It works, but it's another thing you have to maintain and reason about. Don't reach for it unless the default strategy is genuinely causing problems.

## Consumer Group Rebalancing: The Necessary Evil

When a consumer joins or leaves a group (or crashes, or takes too long to send a heartbeat), Kafka triggers a rebalance. During a rebalance, *no consumer in the group can read messages*. The world stops.

The default "eager" rebalance protocol revokes all partition assignments, then reassigns them. This means even partitions that don't change hands get a pause. For a group with 50 consumers, this can mean seconds or even minutes of zero processing.

### Cooperative Sticky Rebalancing

Since Kafka 2.4, you can use the `CooperativeStickyAssignor`. Instead of revoking everything, it only migrates the partitions that need to move. The consumers that keep their partitions never stop processing.

```properties
spring.kafka.consumer.properties.partition.assignment.strategy=\
  org.apache.kafka.clients.consumer.CooperativeStickyAssignor
```

This is a massive improvement. There's really no reason not to use it unless you're stuck on ancient client versions. The caveat: you can't mix eager and cooperative assignors in the same group. If you're migrating, you need to do a rolling update through the `CooperativeStickyAssignor` first.

### Static Group Membership

For even less rebalancing pain, static group membership lets you assign a `group.instance.id` to each consumer. When a consumer with a static ID disconnects briefly, Kafka doesn't immediately trigger a rebalance - it waits for `session.timeout.ms` to expire. When the consumer reconnects with the same ID, it gets its old partitions back without a rebalance.

```properties
spring.kafka.consumer.properties.group.instance.id=order-processor-0
spring.kafka.consumer.properties.session.timeout.ms=60000
```

This is perfect for rolling deployments. Your consumer goes down for 30 seconds during a deploy, comes back up, and picks up exactly where it left off. No rebalance, no duplicate processing window, no downtime.

## Partition Count: The Decision You Can't Undo

Adding partitions to a topic is easy. But if you're using key-based partitioning, adding partitions changes the key-to-partition mapping. Messages for the same key will suddenly land on different partitions than they did before. If your consumers have local state (like Kafka Streams state stores), that state is now in the wrong place.

My rule of thumb: start with more partitions than you think you need. The overhead of empty partitions is tiny compared to the pain of repartitioning a production topic. For most of our services, we use 12-24 partitions per topic, even if we only have 3-4 consumers today.

## The Partition Count to Consumer Ratio

There's no magic formula, but here's what's worked for us:

- **Start with 2-3x your expected consumer count.** This gives you room to scale consumers without changing the topic.
- **Monitor consumer lag per partition.** If one partition consistently has higher lag, you have a hot partition problem, not a "need more partitions" problem.
- **Don't go crazy.** Each partition has overhead on the broker (file handles, memory for index segments, replication traffic). Thousands of partitions per broker is asking for trouble.

The goal is to find the sweet spot where you have enough parallelism for your peak load, without creating so many partitions that broker overhead becomes a problem. For most teams, that number is somewhere between 6 and 50 per topic.
