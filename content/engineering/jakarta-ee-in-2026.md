+++
title = "Jakarta EE in 2026: Still Relevant or Just Nostalgic?"
date = 2025-10-05
[taxonomies]
tags = ["jakarta-ee", "java", "opinion"]
+++

Jakarta EE is the cockroach of the Java ecosystem. I mean that as a compliment. Every year someone writes a "Jakarta EE is dead" blog post, and every year Jakarta EE is still running in production at banks, insurance companies, airlines, and government agencies. It survived the J2EE bloat era, the Spring revolution, the microservices hype cycle, and the cloud-native wave. It's still here.

The question isn't whether Jakarta EE exists. It's whether you should choose it for a new project in 2026.

## The javax to jakarta Migration

Let's get this out of the way. The biggest practical change in recent Jakarta EE history is the namespace migration from `javax.*` to `jakarta.*`. Oracle donated Java EE to the Eclipse Foundation but kept the `javax` trademark, so everything had to be renamed.

```java
// Before (Java EE / Jakarta EE 8)
import javax.persistence.Entity;
import javax.ws.rs.GET;
import javax.inject.Inject;
import javax.servlet.http.HttpServletRequest;

// After (Jakarta EE 9+)
import jakarta.persistence.Entity;
import jakarta.ws.rs.GET;
import jakarta.inject.Inject;
import jakarta.servlet.http.HttpServletRequest;
```

It's a find-and-replace. In theory. In practice, every library in your dependency tree that references `javax.*` needs to be updated to a version that uses `jakarta.*`. If you're using a library that's abandoned and still references `javax.servlet`, you have a problem.

For large enterprise applications with hundreds of dependencies, this migration is a multi-month project. I've seen teams stuck on Jakarta EE 8 because one critical library hasn't migrated. The migration tooling (Eclipse Transformer, OpenRewrite recipes) helps but doesn't solve everything.

## CDI: Dependency Injection Done Differently

Contexts and Dependency Injection (CDI) is Jakarta EE's DI framework. It's conceptually similar to Spring DI but with different conventions:

```java
@ApplicationScoped
public class OrderService {

    @Inject
    private OrderRepository orderRepository;

    @Inject
    private PaymentGateway paymentGateway;

    @Inject
    private Event<OrderCreatedEvent> orderCreatedEvent;

    public Order createOrder(CreateOrderCommand command) {
        Order order = Order.create(command);
        PaymentResult payment = paymentGateway.charge(
            command.customerId(), order.totalAmount());

        order.markAsPaid(payment.transactionId());
        Order saved = orderRepository.save(order);

        orderCreatedEvent.fire(new OrderCreatedEvent(saved.id()));
        return saved;
    }
}
```

CDI has features that Spring didn't have for years (and some it still doesn't match):

**Events**: CDI's `Event<T>` is a built-in, type-safe event system. Fire an event, and any `@Observes` method matching the type receives it. No event bus configuration needed.

```java
public void onOrderCreated(@Observes OrderCreatedEvent event) {
    // react to order creation
}

public void onOrderCreatedAsync(@ObservesAsync OrderCreatedEvent event) {
    // async handler
}
```

**Interceptors and decorators**: CDI separates cross-cutting concerns cleanly:

```java
@Logged
@Interceptor
@Priority(Interceptor.Priority.APPLICATION)
public class LoggingInterceptor {

    @AroundInvoke
    public Object logMethod(InvocationContext context) throws Exception {
        logger.info("Calling: " + context.getMethod().getName());
        return context.proceed();
    }
}
```

**Producers**: factory methods for complex bean creation:

```java
@ApplicationScoped
public class DatabaseProducer {

    @Produces
    @ApplicationScoped
    public DataSource createDataSource(Configuration config) {
        HikariConfig hikari = new HikariConfig();
        hikari.setJdbcUrl(config.getDatabaseUrl());
        return new HikariDataSource(hikari);
    }
}
```

CDI is well-designed. The spec is clean. But Spring's ecosystem (autoconfiguration, starters, community) is so much larger that CDI's technical merits rarely win the tooling argument.

## JAX-RS: REST Done the Standard Way

JAX-RS is the Jakarta EE standard for REST APIs:

```java
@Path("/orders")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
@ApplicationScoped
public class OrderResource {

    @Inject
    private OrderService orderService;

    @GET
    @Path("/{id}")
    public Response getOrder(@PathParam("id") String id) {
        return orderService.findById(id)
            .map(order -> Response.ok(order).build())
            .orElse(Response.status(Status.NOT_FOUND).build());
    }

    @POST
    public Response createOrder(@Valid CreateOrderRequest request) {
        Order order = orderService.createOrder(request.toCommand());
        return Response.status(Status.CREATED)
            .entity(order)
            .header("Location", "/orders/" + order.id())
            .build();
    }
}
```

JAX-RS with reactive (Jakarta EE 10+ with JAX-RS 3.1):

```java
@GET
@Path("/stream")
@Produces(MediaType.SERVER_SENT_EVENTS)
public void streamOrders(@Context SseEventSink sink, @Context Sse sse) {
    orderService.streamAll().forEach(order -> {
        OutboundSseEvent event = sse.newEventBuilder()
            .name("order")
            .data(Order.class, order)
            .build();
        sink.send(event);
    });
}
```

JAX-RS is fine. Spring MVC is fine. They do the same thing with different annotations. The choice is usually ecosystem-driven, not technical.

## MicroProfile: Jakarta EE for Microservices

MicroProfile extends Jakarta EE with patterns needed for microservices:

```java
// Config injection
@Inject
@ConfigProperty(name = "order.max-items", defaultValue = "100")
private int maxOrderItems;

// Fault tolerance
@Retry(maxRetries = 3, delay = 1000)
@CircuitBreaker(requestVolumeThreshold = 4, failureRatio = 0.5)
@Timeout(2000)
public Order fetchOrderFromRemote(String id) {
    return remoteOrderClient.get(id);
}

// Health check
@Readiness
@ApplicationScoped
public class DatabaseHealthCheck implements HealthCheck {

    @Override
    public HealthCheckResponse call() {
        return HealthCheckResponse.named("database")
            .status(isDatabaseReachable())
            .build();
    }
}

// Metrics
@Counted(name = "orders_created_total", description = "Total orders created")
@Timed(name = "order_creation_duration", description = "Time to create an order")
public Order createOrder(CreateOrderCommand command) {
    // business logic
}

// REST client
@RegisterRestClient(configKey = "customer-api")
@Path("/customers")
public interface CustomerClient {

    @GET
    @Path("/{id}")
    Customer getCustomer(@PathParam("id") String id);
}
```

MicroProfile's Config, Fault Tolerance, Health, Metrics, and REST Client specs are direct equivalents of Spring Boot's configuration, Resilience4j, Actuator health, Micrometer, and RestClient/Feign. The APIs are clean and standardized.

The advantage: MicroProfile specs are implemented by Quarkus, Open Liberty, Payara, and WildFly. Your code is portable across runtimes. In theory.

In practice, you pick one runtime and stay with it. The portability is nice for vendor negotiations but rarely exercised.

## Where Jakarta EE Fits in 2026

### Where it makes sense:

**Existing enterprise applications.** If you have a WildFly/Payara/Open Liberty application running in production, modernizing within the Jakarta EE ecosystem (migrating to the latest spec, adopting MicroProfile) is lower risk than rewriting in Spring Boot.

**Regulated industries.** Some organizations have approved technology lists. Jakarta EE on a certified app server is often on that list. Spring Boot might not be. Certification matters when auditors are involved.

**Quarkus.** Here's the irony: Quarkus uses Jakarta EE APIs (CDI, JAX-RS, JPA) with a revolutionary runtime. If you choose Quarkus, you're writing Jakarta EE code. Quarkus is the best thing to happen to Jakarta EE because it proves the APIs are good even if the traditional app server model is dead.

### Where it doesn't make sense:

**Greenfield projects without constraints.** Spring Boot's ecosystem, tooling, and community are larger. Spring Initializr gets you started in 30 seconds. The hiring pool is bigger. Unless you have a specific reason to choose Jakarta EE, Spring Boot is the default for good reasons.

**Microservices with Kubernetes.** Traditional Jakarta EE app servers (WildFly, Payara) are heavyweight for containers. An 800MB Docker image that takes 30 seconds to start doesn't belong in Kubernetes. Quarkus (which uses Jakarta EE APIs) solves this, but at that point you're choosing Quarkus, not traditional Jakarta EE.

## The Honest Opinion

Jakarta EE's APIs are well-designed. CDI is elegant. JAX-RS is clean. MicroProfile fills the microservices gaps. The specifications are thoughtful and standardized.

But the traditional deployment model (WAR/EAR files deployed to an application server) is an anachronism. The world moved to fat JARs, containers, and cloud-native runtimes. Jakarta EE's relevance in 2026 is primarily through Quarkus and lightweight runtimes like Open Liberty, which take the good APIs and discard the heavyweight infrastructure.

If someone asks me whether Jakarta EE is still relevant, my answer is: the APIs are. The deployment model isn't. Use the APIs (via Quarkus or a modern runtime) and leave the application server in the previous decade where it belongs.

And if you're maintaining a legacy Jakarta EE application, you have my sympathy and my respect. That code probably processes more real transactions daily than any conference speaker's Kubernetes demo. The unglamorous work of keeping it running matters more than anyone gives it credit for.
