+++
title = "JMS in 2026: Still Relevant, Still Misunderstood"
date = 2026-02-18
[taxonomies]
tags = ["jms", "messaging", "java"]
+++

I once told a colleague I was debugging a JMS issue and they looked at me like I'd said I was maintaining a COBOL program. Which, fair enough, I've done that too. But here's the thing: JMS is everywhere in enterprise Java. It's in your banks, your telecoms, your insurance companies. It's behind the scenes in systems that process millions of transactions a day. And in 2026, it's still misunderstood by people who think "just use Kafka" is a complete architectural opinion.

## Why JMS Won't Die

JMS is a specification, not an implementation. ActiveMQ, Artemis, IBM MQ, Solace, TIBCO EMS - they all speak JMS. This means you can swap brokers without rewriting your application code. In practice, everyone uses vendor-specific features that break portability, but the *idea* is sound, and the core API is stable enough that code written in 2010 still compiles and runs today.

Try saying that about your favorite JavaScript framework.

## Batching for Performance

One of the most underused features of JMS is session-level batching with `SESSION_TRANSACTED` mode. Instead of acknowledging each message individually, you process a batch and commit the session:

```java
Session session = connection.createSession(true, Session.SESSION_TRANSACTED);
MessageConsumer consumer = session.createConsumer(queue);

int batchSize = 50;
int count = 0;

while (count < batchSize) {
    Message msg = consumer.receive(1000);
    if (msg == null) break;
    processMessage(msg);
    count++;
}

session.commit(); // acknowledges all messages in one go
```

This reduces round trips to the broker dramatically. On a system I worked on last year, switching from individual acks to batched commits with a batch size of 100 improved throughput by roughly 3x. The tradeoff is that if processing fails mid-batch, you roll back and reprocess everything. So keep your batch sizes reasonable and your processing idempotent. (You're seeing a theme here.)

## Message Durability for Topics

JMS topics have two subscriber modes that people constantly conflate:

**Non-durable:** The subscriber only receives messages while connected. Disconnect, and messages published in the meantime are gone.

**Durable:** The broker retains messages for the subscriber even when disconnected. The subscriber catches up when it reconnects.

The gotcha is that durable subscriptions require a unique combination of `clientID` and `subscriptionName`. In a clustered environment where multiple instances of the same service are running, this causes conflicts:

```java
// This breaks with multiple instances - only one can connect with this clientID
connectionFactory.setClientID("inventory-service");

// JMS 2.0 shared durable subscriptions fix this
MessageConsumer consumer = session.createSharedDurableConsumer(
    topic, "inventory-subscription");
```

JMS 2.0's shared durable subscriptions solved a problem that had been causing pain for over a decade. If your broker supports JMS 2.0, use shared subscriptions. If it doesn't, virtual topics (ActiveMQ) or subscription queues (Solace) are your escape hatch.

## JMS Transactions with Spring

Spring's `JmsTransactionManager` integrates JMS sessions with Spring's `@Transactional` annotation. This is straightforward for JMS-only transactions:

```java
@Bean
public JmsTransactionManager jmsTransactionManager(ConnectionFactory cf) {
    return new JmsTransactionManager(cf);
}

@Transactional("jmsTransactionManager")
public void processAndForward(Order order) {
    // receive from input queue (implicit in listener)
    // do some processing
    jmsTemplate.convertAndSend("output.queue", transform(order));
    // commit receives the input message AND sends the output message atomically
}
```

Where it gets tricky is when you need JMS *and* database transactions together. You have three options:

1. **XA transactions (JTA):** Distributed two-phase commit across JMS and the database. Correct but slow and operationally painful. You need a transaction manager like Atomikos or Narayana.
2. **Best-effort 1PC:** Spring's `ChainedTransactionManager` commits the JMS transaction first, then the database. If the DB commit fails, you get an inconsistency. Faster than XA, less correct.
3. **Outbox pattern:** Skip the dual write entirely. Write to the database (including an outbox table) and let a separate process publish to JMS. This is what I actually recommend.

If someone tells you XA transactions are the only correct approach, ask them how their last two-phase commit deadlock investigation went.

## Message Driven POJOs

Spring's `@JmsListener` is essentially a Message Driven POJO - the spiritual successor to EJB's MDBs without the container overhead. The framework handles connection management, session creation, threading, and error recovery:

```java
@JmsListener(destination = "orders.input",
             concurrency = "5-20",
             containerFactory = "jmsListenerContainerFactory")
public void handleOrder(Order order) {
    orderService.process(order);
}
```

The `concurrency = "5-20"` parameter creates between 5 and 20 consumer threads, scaling based on load. This is one of those settings that looks innocuous but matters enormously. Too few threads and you're leaving throughput on the table. Too many and you're overwhelming your downstream services or database connection pool.

I've found that setting the max concurrency to match your database connection pool size (minus a buffer for non-JMS operations) is a reasonable starting point. If your pool has 25 connections, max concurrency of 20 gives you room to breathe.

## AMQP Request/Response Pattern

JMS has a built-in request/response mechanism using `replyTo` headers and temporary queues. It's clunky but it works:

```java
// Requester
Message request = session.createTextMessage(payload);
TemporaryQueue replyQueue = session.createTemporaryQueue();
request.setJMSReplyTo(replyQueue);
request.setJMSCorrelationID(UUID.randomUUID().toString());

producer.send(requestQueue, request);

MessageConsumer replyConsumer = session.createConsumer(replyQueue);
Message reply = replyConsumer.receive(5000); // 5s timeout
```

The temporary queue is created per-request and cleaned up when the session closes. This works for low-throughput scenarios, but at scale, creating and destroying temporary queues adds broker overhead.

A better approach for high-throughput request/response: use a shared reply queue with correlation IDs. Each requester listens on the same reply queue but filters by its correlation ID. Spring's `JmsTemplate.sendAndReceive()` handles this for you, though the default implementation still uses temporary queues under the hood.

For what it's worth, if you need request/response at scale, you probably want HTTP or gRPC, not JMS. Messaging is at its best when the caller doesn't need an immediate answer.

## Compared to Modern Alternatives

Let's be honest about where JMS stands:

| | JMS | Kafka | Solace | RabbitMQ |
|---|---|---|---|---|
| Message replay | No (consumed = gone) | Yes (offset-based) | Replay queues | No |
| Ordering guarantee | Per queue/session | Per partition | Per queue | Per queue |
| Consumer groups | Shared subscriptions (JMS 2.0) | Native | Native | Competing consumers |
| Throughput ceiling | Thousands/sec | Millions/sec | Hundreds of thousands/sec | Tens of thousands/sec |
| Protocol | JMS API (Java only) | Binary (multi-language) | SMF/AMQP/MQTT/JMS | AMQP 0-9-1 |

JMS's biggest limitation is that it's Java-only. In a polyglot world, that's a real constraint. Its biggest strength is that every enterprise Java developer has seen it, the tooling is mature, and it integrates with Spring like they were born together.

## The Bottom Line

JMS in 2026 is like JDBC in 2026 - it's not exciting, it's not what you'd pitch in a tech talk, but it's the foundation layer that an enormous amount of production software depends on. If you're working in enterprise Java, you'll encounter it. Understanding it properly - batching, durability, transactions, the sharp edges - is the difference between a system that works and a system that works until it doesn't.

And when someone tells you JMS is dead, ask them what their bank runs on. Then watch their face.
