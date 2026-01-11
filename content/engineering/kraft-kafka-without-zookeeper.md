+++
title = "KRaft: Kafka Finally Broke Up with ZooKeeper"
date = 2026-01-11
[taxonomies]
tags = ["kafka", "infrastructure"]
+++

ZooKeeper has been the most reluctant participant in Kafka deployments since the beginning. You'd set up your beautiful Kafka cluster, and then you'd also need to babysit a ZooKeeper ensemble: a completely separate distributed system with its own failure modes, its own operational quirks, and its own tendency to ruin your weekends.

With Kafka 4.0, ZooKeeper support is officially gone. KRaft (Kafka Raft) is the only metadata management mode. If you haven't migrated yet, the clock is ticking.

## What KRaft Actually Is

KRaft replaces ZooKeeper with an internal Raft-based consensus protocol for managing Kafka's metadata. Instead of storing partition assignments, broker registrations, and topic configurations in an external ZooKeeper ensemble, Kafka now manages all of this internally using a set of controller nodes.

The controller quorum is a group of Kafka brokers (or dedicated controller nodes) that participate in Raft consensus. One of them is the active controller; the others are hot standbys. Metadata is stored in an internal topic (`__cluster_metadata`) and replicated via Raft. When the active controller fails, a standby takes over almost instantly.

The key insight: Kafka no longer has a split-brain dependency. The thing managing Kafka's metadata *is* Kafka. One operational surface instead of two.

## What Changes Operationally

### Deployment Topology

In ZooKeeper mode, you had Kafka brokers and ZooKeeper nodes. In KRaft mode, you have:

- **Combined mode:** Brokers that also serve as controllers. Simpler to deploy, fine for small to medium clusters.
- **Dedicated controllers:** Separate nodes that only handle metadata. Better for large clusters where you don't want metadata operations competing with broker I/O.

For our environments, we run combined mode for development and staging, and dedicated controllers in production. The dedicated controller nodes are tiny - they barely use any resources since they're not handling data traffic.

### Configuration

The broker config changes are straightforward:

```properties
# KRaft mode
process.roles=broker,controller    # combined mode
# process.roles=controller         # dedicated controller
# process.roles=broker             # dedicated broker

node.id=1
controller.quorum.voters=1@controller1:9093,2@controller2:9093,3@controller3:9093
controller.listener.names=CONTROLLER
```

No more `zookeeper.connect`. No more worrying about ZooKeeper session timeouts causing phantom broker deregistrations. No more debugging ZooKeeper garbage collection pauses that cascade into Kafka instability.

### Faster Failover

This was the improvement I was most excited about, and it delivered. In ZooKeeper mode, controller failover could take 10-30 seconds depending on session timeout configuration. The new controller had to read the entire metadata state from ZooKeeper, which got slower as the cluster grew.

In KRaft mode, failover is sub-second in most cases. The standby controllers already have the metadata replicated locally. There's no cold-start loading phase.

For a cluster with 500+ topics and thousands of partitions, this is the difference between a blip and an outage.

### Cluster Metadata Snapshots

KRaft introduces metadata snapshots to prevent the `__cluster_metadata` topic from growing unbounded. Periodically, controllers write a snapshot of the current metadata state, and older log segments are cleaned up. This is analogous to ZooKeeper's snapshot mechanism, but integrated into Kafka's own storage.

## ISR and Replication: A Quick Refresher

Since we're talking about Kafka internals, let's cover the ISR (In-Sync Replica) mechanism that KRaft continues to manage.

Every partition has a leader and a set of follower replicas. The ISR is the subset of replicas that are "caught up" with the leader. A replica falls out of the ISR if it falls behind by more than `replica.lag.time.max.ms` (default 30 seconds).

When a producer writes with `acks=all`, the write is only acknowledged when all ISR members have the data. If a replica is out of the ISR, it doesn't block writes. This is how Kafka balances durability and availability.

The replication factor determines how many copies of each partition exist. A replication factor of 3 means 3 copies across 3 different brokers. Combined with `min.insync.replicas=2`, you can tolerate one broker failure without data loss and without write interruption.

KRaft doesn't change this mechanism, but it manages it more efficiently. The controller tracks ISR changes faster, which means leader election after a broker failure is quicker.

## The Migration Path

If you're still running ZooKeeper mode, here's the migration path:

1. **Upgrade to Kafka 3.6 or 3.7** (the last versions supporting both modes)
2. **Run `kafka-metadata.sh`** to prepare the migration
3. **Enable KRaft controllers** alongside existing ZooKeeper
4. **Migrate broker by broker** using `kafka-metadata.sh` migration commands
5. **Verify** everything is healthy
6. **Decommission ZooKeeper**

The tooling has gotten much better since the early KRaft releases. The migration is well-documented and, in my experience, less painful than upgrading between major ZooKeeper versions used to be.

One important note: the migration is one-way. Once you've migrated to KRaft, you can't go back to ZooKeeper. Make sure you've tested thoroughly in a non-production environment first.

## Kafka 4.0 Features Worth Knowing

Since KRaft is now the baseline, Kafka 4.0 ships with several features that depend on it:

- **Tiered storage (GA):** Offload old log segments to S3 or Azure Blob Storage. Reduces broker disk requirements significantly.
- **JBOD improvements:** Better support for brokers with multiple disks, with per-disk failure handling.
- **Improved quotas:** More granular client quota management through the controller.

Tiered storage is the one I'm most interested in. We've been managing Kafka disk space like it's 2015, with aggressive retention policies and manual disk expansion. Being able to tier cold data to blob storage while keeping hot data on local SSDs is going to simplify our operations considerably.

## Was It Worth It?

Absolutely. Running ZooKeeper alongside Kafka was always an operational tax. It was a separate system to monitor, backup, upgrade, and debug. Every Kafka outage investigation started with "is ZooKeeper healthy?" and the answer was often "sort of."

KRaft eliminates that tax. Kafka is now a self-contained system. The operational surface is smaller, failover is faster, and there's one less thing to keep running. For anyone still on ZooKeeper mode: start planning your migration. The sooner you do it, the sooner you stop maintaining two distributed systems instead of one.
