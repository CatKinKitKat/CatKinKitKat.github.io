+++
title = "Spring Modulith: The Modular Monolith Done Right"
date = 2026-03-05
[taxonomies]
tags = ["spring-boot", "architecture", "microservices"]
+++

I've spent several articles talking about microservices patterns, and now I'm going to tell you something that might seem contradictory: for most teams, a modular monolith is the better starting point. Not because microservices are bad, but because most teams aren't ready for them, and a modular monolith gives you most of the organizational benefits without the distributed systems tax.

Spring Modulith makes this approach first-class in the Spring ecosystem. And after using it on two projects, I'm convinced it deserves more attention than it gets.

## What Is a Modular Monolith?

A modular monolith is a single deployable artifact with strict internal module boundaries. Unlike a traditional monolith where everything can call everything, modules have explicit public APIs and private internals. Think of it as microservices boundaries without the network.

```
my-application/
  src/main/java/com/example/
    order/               # Order module
      OrderService.java           # Public API
      OrderController.java        # Public API
      internal/
        OrderProcessor.java       # Internal - not accessible from other modules
        OrderValidator.java       # Internal
    inventory/           # Inventory module
      InventoryService.java       # Public API
      internal/
        StockCalculator.java      # Internal
    shipping/            # Shipping module
      ShippingService.java        # Public API
      internal/
        CarrierSelector.java      # Internal
```

Spring Modulith detects these modules by convention. Each top-level package under the main application package is a module. Classes in the root of the module package are the public API. Classes in sub-packages (like `internal`) are module-internal and shouldn't be accessed by other modules.

## Module Boundaries: Verified at Test Time

Here's the killer feature: Spring Modulith verifies module boundaries in your tests. If the order module directly accesses an internal class from the inventory module, the test fails.

```java
@Test
void verifyModuleBoundaries() {
    ApplicationModules modules = ApplicationModules.of(MyApplication.class);
    modules.verify();
}
```

This single test catches architectural violations that would otherwise silently accumulate until your "modular" monolith is a ball of mud. It's like ArchUnit but integrated with Spring's component model, so it understands dependency injection, events, and Spring-specific patterns.

```java
@Test
void documentModules() {
    ApplicationModules modules = ApplicationModules.of(MyApplication.class);
    // Print module structure and dependencies
    modules.forEach(System.out::println);

    // Generate documentation
    new Documenter(modules)
        .writeModulesAsPlantUml()
        .writeIndividualModulesAsPlantUml();
}
```

The documentation generation creates PlantUML diagrams showing module dependencies. When you can visualize which modules depend on which, architectural discussions become grounded in reality instead of opinion.

## Application Events: The Decoupling Mechanism

In a microservices architecture, services communicate through messages (Kafka, RabbitMQ). In a modular monolith, modules communicate through application events. Same principle, zero infrastructure.

```java
// Order module publishes an event
@Service
public class OrderService {

    private final ApplicationEventPublisher events;

    @Transactional
    public Order createOrder(CreateOrderRequest request) {
        Order order = orderRepository.save(new Order(request));

        events.publishEvent(new OrderCreated(
            order.getId(),
            order.getCustomerId(),
            order.getItems(),
            order.getTotal()
        ));

        return order;
    }
}

// The event - in the order module's public API
public record OrderCreated(
    String orderId,
    String customerId,
    List<OrderItem> items,
    BigDecimal total
) {}
```

```java
// Inventory module listens for the event
@Service
public class InventoryReservationHandler {

    @ApplicationModuleListener
    public void onOrderCreated(OrderCreated event) {
        for (OrderItem item : event.items()) {
            inventoryRepository.reserveStock(item.productId(), item.quantity());
        }
    }
}

// Shipping module also listens
@Service
public class ShippingPreparationHandler {

    @ApplicationModuleListener
    public void onOrderCreated(OrderCreated event) {
        shippingService.prepareShipment(event.orderId(), event.items());
    }
}
```

`@ApplicationModuleListener` is Spring Modulith's event listener annotation. It's transactional by default and integrated with the module system. The order module doesn't know about inventory or shipping. It just publishes an event. This is exactly the same decoupling you'd get with Kafka, but with in-process delivery, strong typing, and zero infrastructure.

## Transaction-Aware Event Publication

Here's where Spring Modulith gets really clever. By default, Spring application events are synchronous; the listener runs in the same thread and transaction as the publisher. If the listener fails, the publisher's transaction rolls back. That's often not what you want.

Spring Modulith provides `@ApplicationModuleListener` which, combined with the event publication registry, gives you reliable asynchronous event delivery:

```java
@Configuration
public class ModulithConfig {
    // Spring Modulith auto-configures this if you have the starter
    // Events are stored in the database as part of the publishing transaction
    // A background process delivers them to listeners
}
```

```yaml
spring:
  modulith:
    events:
      republish-outstanding-events-on-restart: true
      jdbc:
        schema-initialization:
          enabled: true
```

The event publication registry stores events in a database table as part of the publisher's transaction. A background thread picks them up and delivers them to listeners. If the application crashes between publishing and delivery, the events are replayed on restart.

This is the outbox pattern built into the framework. No Debezium, no Kafka Connect, no separate infrastructure. The events live in your application database.

## Testing Modules in Isolation

Spring Modulith lets you bootstrap only the module you're testing, with its dependencies mocked or stubbed. This makes integration tests faster and more focused.

```java
@ApplicationModuleTest
class OrderModuleTest {

    @Autowired
    private OrderService orderService;

    @MockBean
    private InventoryService inventoryService;

    @Test
    void createOrder_publishesEvent() {
        // Only the Order module is bootstrapped
        CreateOrderRequest request = new CreateOrderRequest(/* ... */);

        Order order = orderService.createOrder(request);

        assertThat(order.getId()).isNotNull();
    }
}
```

`@ApplicationModuleTest` bootstraps only the beans within the order module (and its declared dependencies). Other modules aren't loaded. This is faster than `@SpringBootTest` and more focused - you're testing the module's behavior in isolation.

For event-based interactions, Spring Modulith provides a `Scenario` API:

```java
@ApplicationModuleTest
class OrderModuleIntegrationTest {

    @Test
    void orderCreation_triggersInventoryReservation(Scenario scenario) {
        scenario.stimulate(() -> orderService.createOrder(testRequest()))
            .andWaitForEventOfType(OrderCreated.class)
            .toArrive()
            .andVerify(event -> {
                assertThat(event.orderId()).isNotNull();
                assertThat(event.total()).isGreaterThan(BigDecimal.ZERO);
            });
    }
}
```

This reads almost like a specification: "When an order is created, an OrderCreated event arrives with a valid order ID and positive total." Tests that read like specs are tests that the team actually understands.

## When a Modulith Beats Microservices

Let me be concrete about when I recommend a modular monolith over microservices.

**Single team.** If one team owns the entire codebase, microservices add coordination overhead with no organizational benefit. Module boundaries within a monolith give you the same code organization without the operational complexity.

**Early in the product lifecycle.** When you're still figuring out domain boundaries, getting them wrong in a monolith means refactoring packages. Getting them wrong in microservices means re-architecting distributed systems. Start with a modulith, extract services when boundaries stabilize.

**Simple deployment model.** If everything deploys together on the same schedule, a single artifact is simpler. One CI pipeline, one deployment, one thing to monitor.

**Transactions across modules.** If your business operations frequently span multiple modules, a monolith gives you database transactions for free. In microservices, you'd need distributed sagas, which are orders of magnitude more complex.

**Shared database.** If modules need to query each other's data (read models, reporting), sharing a database within a monolith is trivial. In microservices, you need event-driven data replication or API composition.

## When to Extract a Service

The modulith doesn't have to stay a monolith forever. Spring Modulith is designed for eventual extraction. The module boundaries and event-based communication are the same patterns you'd use in microservices.

Extract a module into a service when:

1. **Scaling requirements diverge.** The search module needs 10x the resources of the order module. Scaling them independently saves money.
2. **Different deployment cadences.** The recommendation engine updates daily with new ML models. The order processing hasn't changed in months. Independent deployment makes sense.
3. **Technology requirements differ.** The analytics module would benefit from Python/Spark. The rest of the system is Java. Extract it.
4. **Team autonomy.** A new team is dedicated to the payment module. They want to own their release cycle.

The extraction path:

```
1. Module communicates only via events (already done in modulith)
2. Replace in-process events with Kafka/messaging
3. Extract module into its own application
4. Deploy independently
```

Because the module already communicates through events and has verified boundaries, the extraction is mechanical rather than architectural. You're changing the delivery mechanism, not the design.

## The Practical Setup

Here's what a production modulith looks like with Spring Modulith:

```xml
<dependency>
    <groupId>org.springframework.modulith</groupId>
    <artifactId>spring-modulith-starter-core</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.modulith</groupId>
    <artifactId>spring-modulith-starter-jpa</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.modulith</groupId>
    <artifactId>spring-modulith-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

```java
@SpringBootApplication
public class MyApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }
}
```
That's it for the application. The module structure is conventional. The boundaries are verified by tests. Events are stored reliably. And the door is always open to extract services when, and only when, you need to.

## Conclusion
## The Honest Assessment

Spring Modulith isn't perfect. The event publication registry adds database overhead. Module boundary verification can be overly strict for some codebases. And the tooling is still maturing compared to the Spring Boot ecosystem at large.

But the tradeoff is overwhelmingly positive. You get architectural guardrails, event-driven communication, reliable event delivery, isolated testing, and a clear extraction path - all within a single deployable artifact. For the majority of projects I work on, this is the right starting architecture.

The microservices can come later. When they come, they'll come from a position of understanding, not guesswork.
, not guesswork.

