+++
title = "Eventual Consistency: Learning to Live with 'Good Enough'"
date = 2026-03-04
[taxonomies]
tags = ["distributed-systems", "patterns", "microservices"]
+++

Strong consistency is a beautiful lie. It tells you that every read returns the most recent write. That all nodes agree on the state of the world at all times. That your distributed system behaves like a single machine. And for a single database on a single server, it's true. The moment you add a second service, a message broker, a cache, or a read replica, that promise evaporates.

I spent the first few years of my career fighting this reality. Then I spent the next few years learning to work with it. Eventual consistency isn't a weakness of distributed systems - it's a fundamental property that you either design for or suffer from.

## What Eventual Consistency Actually Means

The formal definition: if no new updates are made, all replicas will eventually converge to the same state. The informal definition: things will be correct... eventually. Probably within milliseconds. Definitely within seconds. Almost certainly within minutes.

The "eventually" is what scares people. But here's the thing: the real world is eventually consistent. When you transfer money between bank accounts, there's a period where the money has left one account but hasn't arrived in the other. When you order a product online, there's a window where the inventory shows "in stock" even though the last item was just purchased. These aren't bugs. They're the inherent latency of distributed processes.

The question isn't "should I use eventual consistency?" It's "am I already using it without realizing it, and am I handling it correctly?"

## Event-Driven Architecture on Modern Java

The modern Java stack is well-suited for event-driven systems that embrace eventual consistency. Here's the architecture I've been building with for the past couple of years:

```
[Spring Boot Service] -> [Outbox Table] -> [Debezium/Polling] -> [Kafka/Solace]
        |                                                              |
        v                                                              v
  [PostgreSQL]                                              [Consumer Services]
                                                                    |
                                                                    v
                                                              [Read Models]
```

Each service owns its data. Changes are captured as events. Other services consume events and update their local state. No service directly queries another service's database. No distributed transactions.

With virtual threads (Project Loom), the async gap has narrowed. You can write straightforward blocking code that handles thousands of concurrent event consumers without the thread pool gymnastics that used to be necessary:

```java
@KafkaListener(topics = "orders", concurrency = "100")
public void handleOrderEvent(OrderEvent event) {
    // With virtual threads, 100 concurrent listeners is trivial
    // Each one can do blocking I/O without worrying about thread exhaustion
    enrichmentService.enrich(event);     // calls external API - blocks, that's fine
    localRepository.updateView(event);    // writes to local DB
}
```

The combination of virtual threads, Spring Boot 3.x, and a good message broker makes event-driven architecture feel natural rather than forced. The ceremony of reactive programming (Mono, Flux, subscribe chains) was necessary before Loom. Now it's optional.

## Eventual Consistency and REST

Here's where it gets interesting: your REST API needs to account for eventual consistency. A user creates a resource via POST, then immediately GETs the resource list. If the read model hasn't caught up, the new resource is missing. The user thinks their action failed and tries again.

### Pattern 1: Accepted, Not Created

For operations that trigger asynchronous processes, return `202 Accepted` instead of `201 Created`. Include a location header or resource ID that the client can poll:

```java
@PostMapping("/orders")
public ResponseEntity<OrderAccepted> placeOrder(@RequestBody PlaceOrderRequest request) {
    String orderId = commandHandler.handle(toCommand(request));
    return ResponseEntity
        .accepted()
        .header("Location", "/orders/" + orderId)
        .body(new OrderAccepted(orderId, "Your order is being processed"));
}
```

This sets the right expectation: the operation was received but isn't complete yet. The client polls `/orders/{id}` until the status progresses.

### Pattern 2: Read-Your-Writes Guarantee

For operations where the user expects immediate feedback, bypass the read model for that specific entity:

```java
@GetMapping("/orders/{id}")
public OrderDto getOrder(@PathVariable String id,
                         @RequestHeader(value = "X-Write-Token", required = false) String writeToken) {
    if (writeToken != null && recentWriteCache.contains(writeToken)) {
        // User just wrote this - read from the write model for freshness
        return commandSideRepository.findById(id).map(this::toDto).orElseThrow();
    }
    // Normal read - use the optimized read model
    return readModelRepository.findById(id).map(this::toDto).orElseThrow();
}
```

The write endpoint returns a token (or the client can use the resource ID as a token within a time window). The read endpoint checks for the token and routes to the write model if present. This gives the user consistency for their own writes while preserving the performance benefits of the read model for everyone else.

### Pattern 3: Version-Based Reads

Include a version number in your responses. When the client writes, it gets back the new version. When it reads, it can specify the minimum version it expects:

```java
@GetMapping("/orders/{id}")
public ResponseEntity<OrderDto> getOrder(
        @PathVariable String id,
        @RequestParam(required = false) Long minVersion) {
    OrderDto order = readModelRepository.findById(id).map(this::toDto).orElseThrow();

    if (minVersion != null && order.getVersion() < minVersion) {
        return ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE)
            .header("Retry-After", "1")
            .build();
    }
    return ResponseEntity.ok(order);
}
```

If the read model hasn't caught up to the expected version, the API tells the client to retry. The client can implement a simple retry loop with backoff.

## Local-First Event-Driven Patterns

One of the most practical patterns I've adopted is "local first" event processing. Instead of immediately publishing events to a broker and then having the same service consume them back (a round-trip that adds latency and broker dependency), process events locally first and publish for external consumers.

```java
@Service
public class OrderService {

    private final OrderRepository orderRepository;
    private final OutboxRepository outboxRepository;
    private final LocalEventProcessor localProcessor;

    @Transactional
    public Order placeOrder(PlaceOrderCommand cmd) {
        Order order = Order.place(cmd);
        orderRepository.save(order);

        OrderPlacedEvent event = new OrderPlacedEvent(order);

        // Process locally - update local read models, trigger local side effects
        localProcessor.handle(event);

        // Publish for external consumers via outbox
        outboxRepository.save(new OutboxEntry(event));

        return order;
    }
}
```

The `localProcessor` updates local read models, caches, or triggers follow-up logic within the same transaction. External consumers (other services) get the event via the outbox and broker. This gives you:

- **Immediate local consistency:** Your service's read model is up-to-date the moment the transaction commits.
- **Reduced broker dependency:** If the broker is down, local processing still works. External propagation catches up when the broker recovers.
- **Lower latency:** No round-trip through the broker for things that only matter locally.

This pattern works especially well in systems where most read queries are served by the same service that handles writes. The eventual consistency is only between services, not within a service.

## The Compensation Pattern

When things go wrong in an eventually consistent system (and they will), you need compensating actions. Unlike a database rollback, you can't undo a published event. You publish a new event that reverses the effect.

```java
@EventHandler
public void on(PaymentFailedEvent event) {
    // The order was already confirmed (optimistically), now we need to undo it
    Order order = orderRepository.findById(event.getOrderId()).orElseThrow();
    order.cancel("Payment failed");
    orderRepository.save(order);

    // Publish compensation event
    eventPublisher.publish(new OrderCancelledEvent(
        order.getId(), "Payment failed after confirmation"));
}
```

This is the saga pattern in miniature. Each step in a multi-service process has a corresponding compensating action. If step 3 fails, steps 2 and 1 are compensated in reverse order. It's more complex than a transaction rollback, but it works across service boundaries where transactions can't reach.

## The Uncomfortable Truth

Eventual consistency requires you to think about failure modes that strong consistency hides. What happens when the event is delayed? What happens when events arrive out of order? What happens when an event is processed twice?

These aren't edge cases. In a system with enough traffic, they're regular occurrences. And the answers aren't always technical. Sometimes the right answer is: "The customer sees a stale order status for up to 3 seconds, and that's acceptable." Or: "If inventory is oversold by one unit, the fulfillment team handles it manually."

Not every inconsistency needs a technical solution. Some need a business process. The important thing is that someone has decided what "good enough" means for each scenario, rather than discovering it in production.

## My Rules of Thumb

After a few years of building eventually consistent systems, here's what I keep coming back to:

**Design for at-least-once delivery.** Assume every event handler might be called twice. Make processing idempotent. This is the single most important rule.

**Keep the consistency window small.** Milliseconds, not minutes. Optimize your event processing pipeline. Fast propagation makes eventual consistency invisible to users in most cases.

**Monitor the gap.** Measure the delay between write and read model convergence. Alert when it exceeds your SLA. The metric is simple: timestamp of the latest event processed by the read model minus the current time.

**Make inconsistency visible.** When users might see stale data, tell them. "Last updated 2 seconds ago" is better than pretending the data is real-time when it isn't.

**Start consistent, relax later.** Begin with synchronous processing and a single database. Introduce eventual consistency when you hit a scaling wall or need to decouple services. It's an optimization, not a starting architecture.

Eventual consistency isn't a compromise. It's a design choice that enables scalability, resilience, and independence between services. The trick is making it intentional rather than accidental. When you choose it deliberately and design for it explicitly, it works beautifully. When it sneaks up on you because you didn't think about your distributed system's consistency model... well, that's how you end up debugging phantom data at 2 AM.

And I've done enough of that for one career.
