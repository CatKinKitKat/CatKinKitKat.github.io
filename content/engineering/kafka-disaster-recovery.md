+++
title = "Kafka Disaster Recovery: The Plans Nobody Tests Until It's Too Late"
date = 2026-01-16
[taxonomies]
tags = ["kafka", "infrastructure", "operations"]
+++

I've worked at places where the Kafka DR plan was a Wiki page written by someone who left the company two years ago. It described a procedure that had never been tested, referencing infrastructure that had since been decommissioned. The plan for what happens when the primary datacenter goes down was, effectively, "panic."

If this sounds familiar, keep reading.

## The Multi-Region Problem

Kafka was designed to run within a single datacenter. Replication between brokers assumes low-latency, high-bandwidth connections. Stretch a Kafka cluster across regions, and you're fighting physics - every produce request with `acks=all` now waits for cross-region replication before acknowledging. Your p99 latency goes from 5ms to 150ms, and your throughput craters.

This means multi-region Kafka almost always involves multiple clusters with some kind of replication between them, not one cluster stretched across regions.

## MirrorMaker 2

MirrorMaker 2 (MM2), built on Kafka Connect, is the standard tool for replicating data between Kafka clusters. It handles topic replication, consumer group offset sync, and configuration sync.

The architecture is straightforward: MM2 runs as a set of Kafka Connect connectors that consume from the source cluster and produce to the target cluster. It renames topics by default (prefixing with the source cluster alias) to avoid infinite replication loops in bidirectional setups.

```properties
# mm2.properties
clusters = primary, dr
primary.bootstrap.servers = kafka-primary:9092
dr.bootstrap.servers = kafka-dr:9092

primary->dr.enabled = true
primary->dr.topics = orders.*, payments.*

# offset sync
primary->dr.emit.checkpoints.enabled = true
primary->dr.sync.group.offsets.enabled = true
```

### What Works

- Topic data replication is reliable and well-tested
- Consumer group offset translation lets consumers resume from approximately the right position after failover
- Configuration is declarative and manageable

### What Doesn't

- **Replication lag is unavoidable.** Even with a fast network, MM2 introduces lag. If your primary cluster dies, the latest messages in flight will be lost. This is asynchronous replication - there's no synchronous option without the latency penalty.
- **Offset translation is approximate.** Consumer group offsets in the DR cluster won't be *exactly* where they were in primary. Expect some message reprocessing after failover. Your consumers need to be idempotent (broken record, I know).
- **Topic renaming is confusing.** The default behavior of prefixing topics with the source alias means your consumers need to know which cluster they're pointing at. You can disable renaming, but then you need to be very careful about bidirectional replication loops.

## Active-Passive vs Active-Active

### Active-Passive

One cluster handles all traffic. The other sits idle, receiving replicated data from MM2. On failover, you switch DNS (or reconfigure clients) to point at the DR cluster.

Pros: Simple to reason about. No conflict resolution needed. DR cluster is a straightforward copy.

Cons: The DR cluster costs money to run and serves zero traffic until disaster strikes. You also don't really know if it works until you need it, unless you test failover regularly (you should).

### Active-Active

Both clusters handle traffic, typically for different regions or tenants. MM2 runs bidirectionally, replicating data between clusters. If one goes down, the other handles all traffic.

Pros: Both clusters serve traffic, so you're getting value from the investment. Failover is simpler because the surviving cluster is already handling requests.

Cons: Conflict resolution. If the same topic receives writes in both clusters, you need to handle potential conflicts. Topic naming with MM2 prefixes gets messy. Consumer group offset management in a bidirectional setup is genuinely complex.

We run active-passive for most of our Kafka workloads. The simplicity is worth the cost of an idle DR cluster. For the few services where we need multi-region active processing, we use region-specific topics to avoid conflicts entirely.

## Cloud-Managed Kafka: The Trade-Offs

### Amazon MSK

Runs actual Apache Kafka, so your existing tooling and knowledge transfer directly. Multi-AZ replication is handled for you within a region. Cross-region DR still requires MM2 or a similar tool.

The catch: MSK's managed updates can be slow, and you're limited in configuration flexibility compared to self-hosted. Scaling (adding brokers) is manual and requires partition reassignment. It's "managed" in the sense that AWS handles the underlying infrastructure, but you still manage Kafka.

### Confluent Cloud

Fully managed, scales automatically, and has built-in cluster linking for multi-region replication (better than MM2 in most ways). Confluent Cloud cluster linking provides lower-latency replication with offset preservation.

The catch: cost. Confluent Cloud is significantly more expensive than self-hosted or MSK at scale. For high-throughput workloads, the bill gets eye-watering fast. Also, vendor lock-in - cluster linking is a Confluent feature, not an Apache Kafka feature.

### Self-Hosted on Kubernetes

Maximum control, maximum operational burden. You manage everything: upgrades, scaling, monitoring, DR. Strimzi operator helps enormously, but you're still the one paged when a broker runs out of disk at 3 AM.

The catch: you need a team that knows both Kafka and Kubernetes deeply. If you have that team, self-hosted gives you the best cost-to-performance ratio. If you don't, you'll spend more time operating the platform than building on it.

## Testing Your DR Plan

Here's the part nobody does: actually testing failover.

Schedule a quarterly DR drill. Simulate primary cluster failure. Fail over consumers and producers to the DR cluster. Verify data integrity. Measure how long it takes. Document what went wrong.

The first time we ran a DR drill, it took four hours to complete a failover that was supposed to take thirty minutes. DNS caching in client applications, hardcoded bootstrap servers that someone forgot to update, MM2 replication lag that was much higher than our monitoring suggested - all things we'd never have found without testing.

Now we do it quarterly, and the last drill completed in under 10 minutes. The difference between "we have a DR plan" and "we've tested our DR plan" is the difference between hoping and knowing.

## The Bottom Line

Kafka disaster recovery is not a feature you enable. It's an operational practice you maintain. Pick your architecture (active-passive for simplicity, active-active for utilization), set up MM2 or cluster linking, accept that some data loss is inevitable with async replication, make your consumers idempotent, and - most importantly - test the whole thing regularly.

The best DR plan is the one you've actually executed before you needed it for real.
