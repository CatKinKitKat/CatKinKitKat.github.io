+++
title = "Kafka as a Message Queue: When It Works and When It Really Doesn't"
date = 2026-01-09
[taxonomies]
tags = ["kafka", "messaging", "architecture"]
+++

At least once a quarter, someone in a meeting says "let's just use Kafka for that" about a workload that would be better served by a traditional message queue. And at least once a quarter, I have the same tired conversation about why Kafka and message queues are fundamentally different things that happen to both move messages around.

Let's settle this.

## Kafka Is Not a Queue

A message queue (RabbitMQ, ActiveMQ, Solace, SQS) is built around the idea that a message gets delivered to *one* consumer and then disappears. That's the core contract. Consumer A and Consumer B are both listening? Only one of them gets the message. Once it's acknowledged, it's gone.

Kafka doesn't work this way. Kafka is a distributed log. Messages are written to partitions, and they stay there regardless of who reads them (until retention policy cleans them up). Every consumer group gets its own pointer (offset) into the log. Two consumer groups both reading the same topic both see every message.

Within a single consumer group, Kafka *does* behave sort of like a queue - each partition is assigned to exactly one consumer. But the mapping is partition-to-consumer, not message-to-consumer. You can't have multiple consumers independently picking messages off the same partition within the same group. This distinction matters more than people think.

## The Problems with Kafka-as-Queue

### Head-of-Line Blocking

In a traditional queue, if one message is poison (fails processing repeatedly), you dead-letter it and move on. The next message in line gets picked up by another consumer.

In Kafka, messages within a partition are processed in order. If message 42 keeps failing, your consumer is stuck. Messages 43 through 10,000 sit there waiting. You have to explicitly handle the failure - skip it, send it to a dead letter topic, retry a fixed number of times - before you can make progress.

This is solvable, but it's work. Spring Kafka's `DefaultErrorHandler` with `DeadLetterPublishingRecoverer` does the job, but you need to configure it and test it. It's not the default behavior.

### Scaling Consumers Independently

Want to add a consumer to handle a spike? With a queue, spin up another consumer and it immediately starts pulling messages. With Kafka, your consumer count is bounded by your partition count. If you have 6 partitions and 6 consumers, adding a 7th does nothing. You'd need to add partitions, and that's a topic reconfiguration that has implications for key-based ordering.

### Per-Message Acknowledgment

Traditional queues let you ack or nack individual messages. Kafka's offset model is positional - committing offset 50 implicitly acknowledges everything up to 50. If you successfully processed messages 47, 48, and 50 but failed on 49, you can't just nack 49. You have to commit 48 and reprocess 49 and 50.

There are workarounds (tracking failures separately, using a dead letter topic), but they add complexity that a real queue handles natively.

## KIP-932: Share Groups

The Kafka community knows about these limitations. KIP-932, which landed as a preview in Kafka 3.7 and is maturing through 4.0, introduces share groups - a new consumer group type that gives you actual queue semantics within Kafka.

With share groups:
- Multiple consumers can independently pull messages from the same partition
- Individual messages can be acknowledged or rejected
- Failed messages can be retried without blocking the partition
- No partition-to-consumer assignment - messages are distributed on demand

This is a big deal. It means you can use Kafka for both log-style consumption (traditional consumer groups) and queue-style consumption (share groups) on the same topic. No need to mirror data to a separate queueing system.

That said, as of early 2026, share groups are still evolving. I wouldn't put them in production for mission-critical workloads just yet, but I'm watching the progress closely. The API is stabilizing, and the community feedback loop is active.

## The Kafka vs Postgres Queue Debate

There's a recurring argument online: "just use Postgres as your queue with `SELECT FOR UPDATE SKIP LOCKED`." And honestly? For some workloads, it's not a terrible idea.

```sql
BEGIN;
SELECT * FROM job_queue
  WHERE status = 'pending'
  ORDER BY created_at
  FOR UPDATE SKIP LOCKED
  LIMIT 1;
-- process the job
UPDATE job_queue SET status = 'completed' WHERE id = ?;
COMMIT;
```

This gives you per-message locking, transactional processing, and you don't need any additional infrastructure. For low-throughput workloads (hundreds to low thousands of messages per second), Postgres-as-a-queue works fine.

Where it breaks down: scale. Postgres wasn't built for this. At high throughput, the lock contention, vacuum overhead, and index bloat will eat you alive. Kafka handles millions of messages per second without breaking a sweat. Postgres handles... a lot fewer than that in queue mode.

My rule: if you're already running Postgres and your throughput is modest, use Postgres. If you need real scale, use a real queue (or Kafka). If you need both pub-sub and queue semantics, Kafka with share groups might eventually be the answer.

## When Kafka Actually Works as a Queue

Despite everything I just said, there are cases where Kafka-as-queue is the right call:

1. **You already have Kafka.** Running a separate RabbitMQ cluster just for queue semantics adds operational overhead. If Kafka's limitations don't bite you, using one system is simpler.

2. **You need replay.** Queues destroy messages after consumption. Kafka retains them. If you need to reprocess yesterday's messages because you deployed a bug, Kafka lets you reset offsets. A queue can't.

3. **You want unified infrastructure.** Some teams use Kafka for event streaming and a queue for task processing. If the task processing can tolerate partition-level ordering and bounded consumer scaling, consolidating to Kafka reduces moving parts.

4. **Your consumer count is predictable.** If you know you'll always have 3-6 consumers and your partition count accommodates that, the scaling limitation doesn't matter.

## When It Absolutely Doesn't

1. **High-fanout task distribution** where you need elastic consumer scaling
2. **Workloads with frequent poison messages** that need per-message nack/retry
3. **Priority queues** (Kafka has no native priority mechanism)
4. **Short-lived messages** where retention is waste (though compact topics help)

## The Honest Answer

Kafka is a streaming platform that can be coerced into queue-like behavior with enough effort. Sometimes that effort is justified. Often it isn't. Know the trade-offs, pick the right tool, and resist the urge to make everything a Kafka problem just because Kafka is already running in your cluster.
