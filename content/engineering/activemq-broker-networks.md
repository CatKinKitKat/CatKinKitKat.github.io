+++
title = "ActiveMQ Broker Networks: Legacy Messaging That Refuses to Die"
date = 2026-02-16
[taxonomies]
tags = ["activemq", "messaging", "legacy"]
+++

Let me tell you about ActiveMQ. Not Artemis, not the shiny new stuff - classic ActiveMQ, the 5.x line that's been running in production longer than some of my colleagues have been programming. I've spent more time than I'd like configuring broker networks for enterprise migrations, and if you're reading this, odds are you're in the same boat.

## Network Connectors: The Glue Between Brokers

ActiveMQ's network of brokers is how you build a distributed messaging topology without Kafka-level infrastructure. The idea is straightforward: brokers connect to each other and forward messages to wherever consumers are listening.

A basic network connector looks like this in `activemq.xml`:

```xml
<networkConnectors>
    <networkConnector name="bridge-to-broker2"
        uri="static:(tcp://broker2:61616)"
        duplex="true"
        decreaseNetworkConsumerPriority="true"
        networkTTL="3"
        dynamicOnly="true" />
</networkConnectors>
```

The `duplex="true"` flag means the connection works in both directions - broker1 forwards to broker2 and vice versa. Without it, you need to configure connectors on both ends, which is twice the XML and twice the opportunities to get it wrong.

`dynamicOnly="true"` is the one I always forget and then remember the hard way. Without it, the broker creates a demand-forwarding consumer for every destination, even ones nobody's listening to. On systems with hundreds of queues, that's a lot of wasted connections.

## Clustering and High Availability

ActiveMQ's HA story is... functional. You have two main options:

**Shared storage (NFS/JDBC):** Multiple brokers point at the same persistent store. Only one is active at a time. When the master dies, a standby grabs the lock and takes over. It works, but your shared storage becomes the single point of failure you were trying to eliminate. NFS in particular has given me some spectacular outages.

**Replicated LevelDB (deprecated):** Don't use this. I'm mentioning it because you'll find it in old configs and wonder if you should keep it. You shouldn't. It was based on ZooKeeper and LevelDB, both of which ActiveMQ has moved away from.

The pragmatic choice for most teams I've worked with: shared JDBC store on a properly managed database. It's boring, the DBAs understand it, and it actually stays up. Point your brokers at a PostgreSQL cluster and move on with your life.

```xml
<persistenceAdapter>
    <jdbcPersistenceAdapter dataSource="#postgres-ds"
        lockKeepAlivePeriod="5000"
        cleanupPeriod="10000" />
</persistenceAdapter>
```

## Durable Subscribers: More Complicated Than They Sound

Durable subscriptions let topic consumers disconnect without losing messages. The broker holds onto messages until the subscriber reconnects. Simple in theory. In a broker network? Not simple at all.

The problem: durable subscriber state is local to a broker. If you have brokers A and B in a network, and a durable subscriber connects to A, broker B doesn't automatically know about it. Messages published to B won't get forwarded to A unless you configure things correctly.

You need `conduitSubscriptions="false"` on your network connector. Without it, the broker network treats multiple consumers on the same destination as a single demand, and your durable subscribers start missing messages in ways that are incredibly annoying to debug.

```xml
<networkConnector name="bridge"
    uri="static:(tcp://broker2:61616)"
    conduitSubscriptions="false"
    duplex="true" />
```

I've spent entire afternoons tracing message loss back to this single flag. If you take one thing from this article, let it be this.

## Virtual Topics: The Escape Hatch

Virtual topics are ActiveMQ's answer to "I want pub/sub semantics but with queue-based consumption." You publish to a topic named `VirtualTopic.Orders`, and consumers subscribe to queues named `Consumer.InventoryService.VirtualTopic.Orders`.

Each consumer group gets its own queue with independent cursors. No durable subscriber weirdness, no shared subscription state to manage. Just queues that happen to get fed by a topic.

```java
// Publisher sends to the virtual topic
jmsTemplate.convertAndSend(
    new ActiveMQTopic("VirtualTopic.Orders"), orderEvent);

// Each service consumes from its own queue
@JmsListener(destination = "Consumer.InventoryService.VirtualTopic.Orders")
public void handleOrder(OrderEvent event) {
    // this consumer group gets its own copy of every message
}
```

In every legacy migration I've done, converting durable subscribers to virtual topics has been one of the highest-value changes. Less state to manage, easier to reason about, and they work properly across broker networks without the `conduitSubscriptions` headache.

## Message Priorities: Don't

ActiveMQ supports message priorities (0-9, with 4 as default). You can enable strict priority ordering per destination:

```xml
<policyEntry queue=">" prioritizedMessages="true" useCache="false"
    expireMessagesPeriod="0" queuePrefetch="1" />
```

In theory, higher priority messages get delivered first. In practice, priorities interact badly with prefetch buffers, network forwarding, and consumer acknowledgment patterns. A consumer with a prefetch of 100 will pull 100 messages in one go, and they'll be processed in whatever order they arrived in that batch, regardless of priority.

If you genuinely need priority processing, use separate queues. `orders.high`, `orders.normal`, `orders.low`. Have consumers drain the high-priority queue first. It's crude, it works, and it doesn't depend on broker internals behaving the way the documentation says they will.

## Performance Tuning: The Hits

After years of tuning ActiveMQ, here's what actually moves the needle:

**Prefetch size:** The default is 1000 for queues and 32766 for topics. For slow consumers, lower it. For fast consumers, keep it high. There's no universal right answer, but if your consumers are doing I/O-heavy processing, a prefetch of 10-50 prevents one consumer from hogging messages.

**Producer flow control:** ActiveMQ throttles producers when memory limits are hit. This is good in theory but causes mysterious hangs in practice. If your producers are blocking for no apparent reason, check the memory limits:

```xml
<systemUsage>
    <memoryUsage><memoryUsage percentOfJvmHeap="70" /></memoryUsage>
    <storeUsage><storeUsage limit="50 gb" /></storeUsage>
    <tempUsage><tempUsage limit="20 gb" /></tempUsage>
</systemUsage>
```

**Async dispatch:** Enable `optimizedDispatch="true"` on your policy entries. It reduces lock contention for high-throughput queues.

**Batch acknowledgment:** Use `optimizeAcknowledge="true"` on the connection URI. It batches acks to the broker, reducing round trips. The tradeoff is that you might lose a few acks on crash, so again - idempotent consumers.

## Monitoring with Jolokia

ActiveMQ exposes JMX beans, and Jolokia wraps them in a REST API. This is how you monitor without setting up a full JMX infrastructure.

```bash
# Queue depth
curl -s "http://admin:admin@localhost:8161/api/jolokia/read/\
org.apache.activemq:type=Broker,brokerName=localhost,\
destinationType=Queue,destinationName=orders/QueueSize"

# Enqueue/dequeue counts
curl -s "http://admin:admin@localhost:8161/api/jolokia/read/\
org.apache.activemq:type=Broker,brokerName=localhost,\
destinationType=Queue,destinationName=orders/EnqueueCount,DequeueCount"
```

Scrape these with Prometheus (there's a JMX exporter, or you can hit Jolokia directly), build a Grafana dashboard, and set alerts on queue depth trends. The metric you care about most is the gap between enqueue and dequeue rates. If that gap is growing, your consumers are falling behind and you need to figure out why before the broker runs out of storage.

## The Honest Assessment

ActiveMQ classic is not a bad piece of software. It's mature, well-understood, and it does what it says. But it's showing its age. The codebase is enormous, the configuration is XML-heavy, and the broker network model has sharp edges that newer systems have smoothed over.

If I were starting from scratch, I wouldn't pick ActiveMQ. But I'm not starting from scratch - I'm migrating systems that already run on it, and understanding how it works is the difference between a smooth migration and a 3 AM incident call. These broker networks will outlive most of us in this industry, and knowing how to keep them running while you plan your escape is a genuinely useful skill.
