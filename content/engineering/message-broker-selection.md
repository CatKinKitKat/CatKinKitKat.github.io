+++
title = "Choosing a Message Broker: A Field Guide for the Indecisive"
date = 2026-02-23
[taxonomies]
tags = ["messaging", "architecture", "patterns"]
+++

I've been in at least a dozen meetings where the topic was "which message broker should we use" and the conclusion was "let's schedule another meeting." Broker selection becomes paralysis when you start from the technology and work backward to the problem. So let's flip it: start from the messaging patterns you need, then pick the broker that does them best.

## The Patterns

Before you can pick a broker, you need to know what kind of messaging you're doing. There are four fundamental patterns, and most systems use a combination.

### Pub/Sub (Publish/Subscribe)

One publisher, many subscribers. Each subscriber gets a copy of every message. This is your event notification pattern - "something happened, anyone who cares should know about it."

The nuances that matter:
- **Filtering:** Can subscribers filter which messages they receive? Kafka uses topic-per-event-type. RabbitMQ uses topic exchanges with routing keys. Solace uses hierarchical topic subscriptions. The filtering mechanism shapes your topic design.
- **Durability:** What happens to messages when a subscriber is offline? Kafka retains everything by default (retention-based). RabbitMQ and ActiveMQ require durable subscriptions. Solace uses replay queues.
- **Ordering:** Pub/sub usually doesn't guarantee ordering across subscribers, but ordering within a single subscriber's stream matters. Kafka guarantees per-partition ordering. Others guarantee per-queue ordering.

### Point-to-Point

One producer, one consumer (or a pool of competing consumers). The message is processed exactly once (well, at-least-once in practice). This is your work queue pattern - "here's a task, someone do it."

Point-to-point is the simplest pattern and every broker handles it well. The differentiator is what happens when processing fails:
- **Dead-letter queues:** Where do failed messages go? RabbitMQ and ActiveMQ have built-in DLQ support. Kafka doesn't - you build it yourself with a separate topic and error handling logic.
- **Retry semantics:** Can the broker automatically retry with backoff? RabbitMQ can with delayed message plugins. Kafka requires application-level retry.
- **Priority:** Can some messages jump the queue? ActiveMQ and RabbitMQ support message priorities. Kafka doesn't. Solace supports priority queues.

### Request/Reply

Synchronous-style communication over asynchronous messaging. The requester sends a message and waits for a response on a reply queue. This is useful when you need the decoupling benefits of messaging but the interaction pattern is still request/response.

Honestly? This pattern is the most contentious. Half the architects I know say it's an anti-pattern ("just use HTTP"). The other half say it's essential for decoupling. My take: request/reply over messaging makes sense when you need the broker's routing or reliability features, or when the request needs to survive network partitions. Otherwise, HTTP or gRPC is simpler.

RabbitMQ has the best built-in support for this pattern (direct reply-to queues). Solace handles it natively. JMS has `replyTo` headers. Kafka technically supports it via the `KafkaReplyingTemplate`, but it's awkward and nobody seems happy using it.

### Fire-and-Forget

The producer sends a message and doesn't care what happens next. No acknowledgment, no response, no tracking. This is your logging, metrics, and audit trail pattern.

Every broker supports this. The question is how much durability you want. For fire-and-forget where message loss is acceptable, you can use non-persistent messages for maximum throughput. For fire-and-forget where messages must eventually be processed, you want persistence with lenient consumer SLAs.

## The Brokers

Now let's talk about the actual options. I'm covering the four I've used in production. There are others (NATS, Pulsar, Redis Streams), but I'm not going to pretend I have production experience with them.

### Kafka

**What it is:** A distributed, persistent commit log.

**Where it shines:**
- Event streaming at massive scale (millions of messages per second)
- Event sourcing and event replay - consumers can re-read from any offset
- Stream processing (Kafka Streams, ksqlDB)
- Audit trails and data pipelines

**Where it doesn't:**
- Traditional work queues (consumer groups are partition-bound, not message-bound)
- Complex routing (one topic, that's your routing)
- Low-latency messaging (partition assignment and rebalancing add latency)
- Small teams - Kafka is operationally heavy, even with managed offerings

**The honest take:** Kafka is phenomenal for event-driven architectures where you need a durable, replayable event log. It's overkill for "send a job to a worker" use cases. I've seen teams adopt Kafka because it's impressive on a resume and then spend months fighting consumer group rebalancing issues for what is essentially a task queue.

### RabbitMQ

**What it is:** A traditional message broker with flexible routing.

**Where it shines:**
- Complex routing patterns (topic, headers, fanout exchanges)
- Work queues with competing consumers
- Request/reply patterns
- Dead-letter queues and message TTL
- Teams that want something that "just works" for moderate scale

**Where it doesn't:**
- Message replay (consumed = gone, unless you build replay yourself)
- Extreme throughput (ceiling is tens of thousands per second per queue)
- Event sourcing

**The honest take:** RabbitMQ is the Toyota Corolla of message brokers. It's not exciting, it doesn't win benchmarks, but it reliably does what most people need. The plugin ecosystem is solid, the management UI is genuinely useful, and the documentation is excellent.

### ActiveMQ (Classic and Artemis)

**What it is:** The veteran enterprise broker.

**Where it shines:**
- Legacy enterprise integration (JMS, AMQP, STOMP, MQTT from one broker)
- Protocol flexibility - speaks basically everything
- Virtual topics (classic) for pub/sub with queue semantics
- Existing WebLogic/JBoss/WildFly integrations

**Where it doesn't:**
- Horizontal scaling (broker networks have sharp edges, as I've written about)
- Modern operational tooling (compared to Kafka or RabbitMQ, monitoring is clunky)
- Throughput at the top end

**The honest take:** If you're inheriting ActiveMQ, learn it well. If you're choosing from scratch... Artemis is a reasonable option, but you'll likely be happier with RabbitMQ or Kafka depending on your pattern. Classic ActiveMQ is maintenance-mode software at this point.

### Solace

**What it is:** Enterprise messaging platform with multi-protocol support and hierarchical topics.

**Where it shines:**
- Financial services and high-performance messaging
- Hierarchical topic subscriptions (subscribe to `orders/>` to get all order subtopics)
- Multi-protocol (MQTT, AMQP, JMS, REST) from one broker
- Event mesh across cloud regions
- Replay queues for event recovery

**Where it doesn't:**
- Open-source community and ecosystem (it's a commercial product, the free tier has limits)
- Developer experience for small projects (the learning curve is steep)
- Cost for small teams

**The honest take:** Solace is the broker I've used most at work, and it's genuinely impressive for enterprise messaging. The hierarchical topic system is something I miss when using other brokers. But it's a commercial product, and the ecosystem is smaller than Kafka's or RabbitMQ's. If your company is already paying for Solace, use it. If you're choosing for a startup, probably not.

## My Decision Framework

When someone asks me which broker to use, I ask three questions:

**1. Do you need event replay?** If yes, Kafka. Nothing else does replay as a first-class feature. If you might need replay later, still probably Kafka - retrofitting replay onto a traditional broker is painful.

**2. Do you need complex routing?** If yes, RabbitMQ or Solace. Kafka's routing is "pick a topic" which is sometimes all you need and sometimes woefully insufficient. If your messages need to be routed based on content, headers, or hierarchical patterns, you want a broker that supports it natively.

**3. What's your scale?** If you're processing fewer than 10,000 messages per second, any broker will do. Pick the one your team knows. If you're processing millions per second, it's Kafka or Solace. RabbitMQ and ActiveMQ will struggle at that scale.

And the meta-question: **What does your team already know?** The best broker is the one your team can operate confidently. A well-operated RabbitMQ cluster beats a poorly-operated Kafka cluster every time. Technology choices are also people choices.

## The Wrong Answer

The only truly wrong answer is picking a broker because of a blog post (including this one) without understanding your specific requirements. I've seen teams pick Kafka for simple task queues and RabbitMQ for event sourcing. Both ended up migrating. Both migrations were expensive.

Understand your messaging patterns first. The broker choice usually becomes obvious after that.
