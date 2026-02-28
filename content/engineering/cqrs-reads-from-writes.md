+++
title = "CQRS: Separating Reads from Writes Without Losing Your Mind"
date = 2026-02-28
[taxonomies]
tags = ["cqrs", "architecture", "patterns"]
+++

CQRS - Command Query Responsibility Segregation - is one of those patterns that sounds like it was named by a committee trying to maximize syllable count. But underneath the enterprise-speak is a surprisingly simple idea: use different models for reading and writing data.

If you've ever added a database view or a denormalized table because your read queries were getting too complex or too slow, congratulations - you've already done a lightweight version of CQRS. The pattern just gives it a name and a structure.

## The Pattern

In a traditional application, the same model serves reads and writes. Your `Order` entity has fields for processing (status, payment info, shipping details) and fields for display (customer name, formatted totals, delivery estimate). The entity becomes a compromise - too complex for simple reads, too constrained for complex writes.

CQRS splits this into two sides:

**Command side (write model):** Handles state changes. Validates business rules. Stores data in a format optimized for consistency and transactional integrity. This is your normalized, relational model.

**Query side (read model):** Serves queries. Stores data in a format optimized for fast reads. Denormalized, pre-computed, shaped exactly for the UI or API consumer. No joins at query time.

The two sides are synchronized through events. When the write model changes, it publishes an event. The read model consumes the event and updates its view.

```
[Command] --> [Write Model / DB] --> [Event] --> [Read Model / DB] --> [Query Response]
```

That's it. That's the whole pattern. Everything else is implementation detail.

## Read Model Projections

Projections are the read-side event handlers that build and maintain your query-optimized views. Each projection listens for relevant events and updates its read model accordingly.

```java
@Component
public class OrderSummaryProjection {

    private final OrderSummaryRepository summaryRepo;

    @EventHandler
    public void on(OrderPlacedEvent event) {
        OrderSummary summary = new OrderSummary();
        summary.setOrderId(event.getOrderId());
        summary.setCustomerName(event.getCustomerName());
        summary.setTotalAmount(event.getTotalAmount());
        summary.setStatus("PLACED");
        summary.setItemCount(event.getItems().size());
        summary.setPlacedAt(event.getTimestamp());
        summaryRepo.save(summary);
    }

    @EventHandler
    public void on(OrderShippedEvent event) {
        OrderSummary summary = summaryRepo.findById(event.getOrderId())
            .orElseThrow();
        summary.setStatus("SHIPPED");
        summary.setTrackingNumber(event.getTrackingNumber());
        summary.setShippedAt(event.getTimestamp());
        summaryRepo.save(summary);
    }

    @EventHandler
    public void on(OrderDeliveredEvent event) {
        OrderSummary summary = summaryRepo.findById(event.getOrderId())
            .orElseThrow();
        summary.setStatus("DELIVERED");
        summary.setDeliveredAt(event.getTimestamp());
        summaryRepo.save(summary);
    }
}
```

The `OrderSummary` table is completely denormalized - no joins needed to serve an "order list" API endpoint. Customer name, total, status, item count, tracking number, all in one row. The query becomes a simple `SELECT * FROM order_summary WHERE customer_id = ? ORDER BY placed_at DESC`.

You can have multiple projections for different query needs:
- `OrderSummaryProjection` for the order list page
- `OrderDetailProjection` for the order detail page (with line items)
- `OrderAnalyticsProjection` for dashboards (aggregated by day, category, region)

Each projection maintains its own read model, shaped for its specific consumer. This is where CQRS pays for itself - your read models are purpose-built, not one-size-fits-all compromises.

## Implementation with Spring

Here's how I've structured CQRS in Spring Boot applications. Nothing exotic - just clear separation of concerns.

### The Write Side

```java
// Command
public record PlaceOrderCommand(String customerId, List<OrderItem> items) {}

// Command handler
@Service
public class OrderCommandHandler {

    private final OrderRepository orderRepo;
    private final ApplicationEventPublisher eventPublisher;

    @Transactional
    public String handle(PlaceOrderCommand cmd) {
        // Business validation
        if (cmd.items().isEmpty()) {
            throw new IllegalArgumentException("Order must have at least one item");
        }

        // Create and persist the aggregate
        Order order = Order.place(cmd.customerId(), cmd.items());
        orderRepo.save(order);

        // Publish domain event
        eventPublisher.publishEvent(new OrderPlacedEvent(
            order.getId(), cmd.customerId(),
            order.getTotal(), cmd.items(), Instant.now()
        ));

        return order.getId();
    }
}
```

### The Read Side

```java
// Query
public record GetOrderSummariesQuery(String customerId, int page, int size) {}

// Query handler
@Service
public class OrderQueryHandler {

    private final OrderSummaryRepository summaryRepo;

    @Transactional(readOnly = true)
    public Page<OrderSummaryDto> handle(GetOrderSummariesQuery query) {
        return summaryRepo.findByCustomerId(
            query.customerId(),
            PageRequest.of(query.page(), query.size(), Sort.by("placedAt").descending())
        ).map(this::toDto);
    }
}
```

### The Controller

```java
@RestController
@RequestMapping("/orders")
public class OrderController {

    private final OrderCommandHandler commandHandler;
    private final OrderQueryHandler queryHandler;

    @PostMapping
    public ResponseEntity<String> placeOrder(@RequestBody PlaceOrderRequest request) {
        String orderId = commandHandler.handle(toCommand(request));
        return ResponseEntity.created(URI.create("/orders/" + orderId)).body(orderId);
    }

    @GetMapping
    public Page<OrderSummaryDto> listOrders(
            @RequestParam String customerId,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        return queryHandler.handle(new GetOrderSummariesQuery(customerId, page, size));
    }
}
```

Notice the read and write paths are completely separate. Different handlers, different repositories, potentially different database connections. The write side uses `OrderRepository` backed by the normalized schema. The read side uses `OrderSummaryRepository` backed by the denormalized view.

### Event Propagation

For single-service CQRS (write model and read model in the same application), Spring's `ApplicationEventPublisher` works fine. Events are handled synchronously within the same transaction by default, or asynchronously with `@Async`.

For multi-service CQRS (read model in a separate service), you need a message broker. Publish events to Kafka or RabbitMQ, and have the read service consume them. This is where the outbox pattern comes in - write the event to an outbox table in the same transaction as the business data, then publish to the broker asynchronously.

## When CQRS Makes Sense

CQRS isn't free. It adds architectural complexity: two models, event propagation, eventual consistency between read and write sides. You need to justify that cost.

**CQRS makes sense when:**

- Read and write workloads have dramatically different scaling needs. If your system handles 100 writes per second but 10,000 reads per second, scaling the read side independently is valuable.
- Read queries require complex joins or aggregations that slow down the write model. Pre-computing these into denormalized views eliminates query-time complexity.
- You're already doing event sourcing. CQRS is the natural complement - events drive the write side, projections build the read side.
- Different consumers need different representations of the same data. API clients want JSON summaries. Analytics wants time-series aggregations. Internal tools want raw detail. Multiple read models serve each consumer optimally.

**CQRS is overkill when:**

- Your application is a standard CRUD app with similar read and write complexity. Adding CQRS to a user management service is like buying a truck to carry groceries.
- Your team is small and the operational overhead of maintaining two models outweighs the benefits. Every projection is code you need to write, test, and keep in sync.
- Read performance is fine with the existing model. If a simple index solves your query performance problem, you don't need a separate read model.
- You're early in the project and the domain isn't well understood. CQRS bakes in assumptions about read and write patterns that are hard to change later. Get the domain right first, then optimize.

## The Eventual Consistency Problem

The elephant in the room: when the write and read models are separate, there's a propagation delay. A user creates an order and immediately queries for it. If the projection hasn't processed the event yet, the order doesn't appear.

Solutions, from simplest to most complex:

1. **Accept it.** Show a confirmation page that doesn't depend on the read model. "Your order has been placed." The list view will catch up in milliseconds.
2. **Read-your-own-writes.** After a write, query the write model directly for that specific entity. Use the read model for list views and search. This is a pragmatic compromise that covers 90% of cases.
3. **Synchronous projections.** Update the read model within the same transaction as the write. This eliminates the delay but couples the two sides and reduces the benefits of CQRS.

In practice, option 1 or 2 is usually sufficient. If you're fighting eventual consistency everywhere in your application, CQRS might not be the right pattern for your domain. Or your propagation latency is too high and you need to optimize the event handling pipeline.

## The Honest Summary

CQRS is a powerful pattern that solves real problems. It's also a pattern that gets cargo-culted into systems where it doesn't belong, turning simple applications into distributed systems puzzles. The key is recognizing when the separation of reads and writes genuinely helps - different scaling needs, complex read models, event-sourced domains - and when it's adding ceremony without value.

Start without CQRS. When you hit a wall where read and write models need to diverge, introduce it. The transition from a traditional model to CQRS is much smoother than the transition from over-engineered CQRS back to simplicity.
