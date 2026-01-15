+++
title = "Quarkus: The Kubernetes-Native Java Framework That Earned My Respect"
date = 2025-09-15
[taxonomies]
tags = ["quarkus", "java", "kubernetes"]
+++

I'm a Spring Boot developer. I've been one for years. My muscle memory types `@RestController` before my brain engages. So when I first looked at Quarkus, it was with the skepticism of someone who's seen a dozen Spring alternatives come and go.

Then I watched a Quarkus service start in 0.8 seconds. My equivalent Spring Boot service took 12 seconds. I wasn't skeptical anymore. I was curious.

## Dev Mode: The Hook

Quarkus dev mode (`mvn quarkus:dev`) is what got me to take it seriously. You change a Java file, hit the endpoint, and the change is live. No restart. No rebuild. Sub-second hot reload.

Spring Boot has DevTools with restart. It's faster than a full restart but still takes 3-5 seconds. Quarkus dev mode is nearly instant because it reloads only the changed classes at the bytecode level, not the entire application context.

Dev mode also gives you a dev UI at `localhost:8080/q/dev` with live database management, configuration browser, Swagger UI, health checks, and a continuous testing panel that reruns affected tests as you code.

It sounds like a gimmick. It's not. The feedback loop difference between "save, wait 5 seconds, test" and "save, test" compounds over a full day of development.

## Why It's Fast

Quarkus does at build time what Spring Boot does at runtime. Classpath scanning, annotation processing, proxy generation, dependency injection wiring - all of it happens during compilation. The result is a pre-built application model that starts almost instantly.

Spring Boot's autoconfiguration is powerful but it pays for that flexibility at startup. It scans the classpath, evaluates conditions, creates proxies, and wires beans - all at runtime, every time the application starts.

Quarkus's approach is opinionated: if you want build-time optimization, the framework needs to know what you're doing at build time. This means some runtime dynamic features (like runtime classpath scanning for custom annotations) don't work the same way. In practice, this rarely matters because Quarkus provides extensions for almost everything.

## Native Image: The Promise and the Reality

Quarkus can compile to a GraalVM native image:

```bash
mvn package -Pnative
```

The result is a standalone binary. No JVM required. Startup time: 20-50ms. Memory usage: 50-80MB for a typical REST service. Compare that to 12 seconds and 300MB for a JVM Spring Boot service.

For serverless and CLI tools, this is transformative. For long-running services, the tradeoff is peak throughput: native images don't have JIT compilation, so CPU-intensive hot paths are slower. In my testing, native images handle 70-85% of the JVM throughput for typical web services. For I/O-bound services (most of what I build), the difference is negligible.

The build time is painful: 3-5 minutes on a good machine, much longer in CI. And some Java features need configuration to work with native image (reflection, dynamic proxies, JNI). Quarkus handles most of this automatically for its extensions, but third-party libraries might need `@RegisterForReflection` annotations or `reflect-config.json` files.

## Kafka Streams on Quarkus

Quarkus has first-class Kafka integration via SmallRye Reactive Messaging:

```java
@ApplicationScoped
public class OrderEventProcessor {

    @Incoming("orders")
    @Outgoing("order-notifications")
    public OrderNotification process(OrderEvent event) {
        // transform event
        return new OrderNotification(event.orderId(), event.status());
    }
}
```

Configuration:

```properties
mp.messaging.incoming.orders.connector=smallrye-kafka
mp.messaging.incoming.orders.topic=order-events
mp.messaging.incoming.orders.value.deserializer=io.quarkus.kafka.client.serialization.JsonbDeserializer

mp.messaging.outgoing.order-notifications.connector=smallrye-kafka
mp.messaging.outgoing.order-notifications.topic=order-notifications
```

The reactive messaging model is different from Spring's `@KafkaListener` but achieves the same thing with less ceremony. The channel-based approach (incoming/outgoing) makes the data flow explicit.

For Kafka Streams specifically, Quarkus has a Kafka Streams extension that exposes the topology builder:

```java
@ApplicationScoped
public class OrderStreamsTopology {

    @Produces
    public Topology buildTopology() {
        StreamsBuilder builder = new StreamsBuilder();
        builder.stream("orders", Consumed.with(Serdes.String(), orderSerde))
            .filter((key, order) -> order.status() == OrderStatus.PAID)
            .mapValues(order -> new ShipmentRequest(order.id(), order.items()))
            .to("shipments", Produced.with(Serdes.String(), shipmentSerde));
        return builder.build();
    }
}
```

## Virtual Threads

Quarkus supports virtual threads with `@RunOnVirtualThread`:

```java
@Path("/orders")
public class OrderResource {

    @GET
    @Path("/{id}")
    @RunOnVirtualThread
    public Order getOrder(@PathParam("id") String id) {
        return orderService.findById(id); // blocking call, runs on virtual thread
    }
}
```

Or configure it globally. The integration is straightforward because Quarkus's reactive architecture was already designed for non-blocking - virtual threads are another option alongside reactive and coroutines.

## Knative Eventing

Quarkus + Knative is the serverless story on Kubernetes. Knative scales to zero, and Quarkus's fast startup (especially native) means scale-from-zero latency is minimal:

```java
@ApplicationScoped
public class OrderEventHandler {

    @Incoming("orders")
    @Acknowledgment(Acknowledgment.Strategy.POST_PROCESSING)
    public void handleOrderEvent(CloudEvent<OrderEvent> event) {
        // process CloudEvent
    }
}
```

Knative Eventing delivers CloudEvents to your Quarkus service via HTTP. The service scales based on event volume. When there are no events, it scales to zero. When events arrive, Knative starts instances, and Quarkus boots in under a second.

## LangChain4j Integration

Quarkus has a LangChain4j extension for AI integration:

```java
@RegisterAiService
public interface OrderAssistant {

    @SystemMessage("You are an order management assistant.")
    @UserMessage("Summarize this order: {order}")
    String summarizeOrder(Order order);
}
```

```java
@Path("/orders")
public class OrderResource {

    @Inject
    OrderAssistant assistant;

    @GET
    @Path("/{id}/summary")
    public String getOrderSummary(@PathParam("id") String id) {
        Order order = orderService.findById(id);
        return assistant.summarizeOrder(order);
    }
}
```

The AI service is a CDI bean. Quarkus handles the API calls, retries, and response parsing. It's the cleanest AI integration I've seen in a Java framework.

## Spring Boot vs Quarkus: The Honest Comparison

**Startup time**: Quarkus wins decisively. JVM: 0.8s vs 12s. Native: 0.05s vs not applicable (Spring Native exists but is less mature).

**Runtime performance**: roughly equivalent for I/O-bound services. Spring Boot has a slight edge for CPU-bound work due to JIT optimization in long-running processes.

**Ecosystem**: Spring Boot wins. More libraries, more tutorials, more Stack Overflow answers, more enterprise adoption.

**Developer experience**: Quarkus dev mode is better than Spring DevTools. Spring Boot's autoconfiguration is more forgiving.

**Native image**: Quarkus wins. Better support, more extensions tested with native, faster builds.

**Hiring**: Spring Boot wins. More developers know it. This matters for consultancies.

**Migration effort**: high. Quarkus uses CDI, not Spring DI. JAX-RS, not Spring MVC (though Quarkus has a Spring compatibility layer). Different testing approach. It's not a drop-in replacement.

My honest take: if I were starting a greenfield project with Kubernetes-first requirements (fast startup, low memory, native image), I'd seriously consider Quarkus. For adding to an existing Spring Boot ecosystem, the migration cost usually doesn't justify the startup time improvement. Both are excellent frameworks for serious Java development.
