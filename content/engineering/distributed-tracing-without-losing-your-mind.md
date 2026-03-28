+++
title = "Distributed Tracing Without Losing Your Mind"
date = 2026-03-28
[taxonomies]
tags = ["observability", "spring-boot", "microservices"]
+++

The moment you split a monolith into microservices, debugging gets harder by an order of magnitude. A request that used to be a single stack trace is now a chain of HTTP calls, Kafka messages, and database queries spread across a dozen services. Something is slow. Which service? Which call? Which step?

Without distributed tracing, the answer is "we don't know, let's check every service's logs and hope the timestamps line up." With distributed tracing, the answer is "look at this trace, the bottleneck is the inventory service's database query at 450ms."

## The Basics

Distributed tracing assigns a unique trace ID to each incoming request and propagates it across every service-to-service call. Each service creates spans - named, timed segments - that represent the work it's doing. The trace collector aggregates everything into a timeline.

The OpenTelemetry standard handles this. Spring Boot 3 has built-in support via Micrometer Tracing and the OpenTelemetry exporter.

## Setting It Up

Add the dependencies:

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-otlp</artifactId>
</dependency>
```

Configure the exporter:

```yaml
management:
  tracing:
    sampling:
      probability: 1.0  # sample everything in dev, reduce in prod
  otlp:
    tracing:
      endpoint: http://tempo:4318/v1/traces
```

That's it for HTTP calls. Spring auto-instruments RestTemplate, WebClient, and Spring MVC. Every outgoing HTTP call propagates the trace context automatically.

## Kafka Is the Hard Part

HTTP propagation is solved. Kafka propagation is where things get interesting, because messages are asynchronous. The trace context needs to travel inside the Kafka message headers.

Spring Kafka's `ProducerInterceptor` and `ConsumerInterceptor` handle this if you configure the tracing observation:

```yaml
spring:
  kafka:
    producer:
      properties:
        interceptor.classes: io.opentelemetry.instrumentation.kafkaclients.v2_6.TracingProducerInterceptor
    consumer:
      properties:
        interceptor.classes: io.opentelemetry.instrumentation.kafkaclients.v2_6.TracingConsumerInterceptor
```

Now when Service A publishes a message and Service B consumes it, both spans appear in the same trace. The gap between them shows how long the message sat in the Kafka topic.

## What to Trace (And What Not To)

Trace everything in development. In production, sample. A `probability` of `0.1` means 10% of traces are captured, which is usually enough to diagnose issues without drowning your trace backend.

For specific slow operations, add custom spans:

```java
@Observed(name = "order.enrichment")
public Order enrichOrder(Order order) {
    // this method gets its own span in the trace
    ExternalData data = externalApi.fetchData(order.getCustomerId());
    return order.withExternalData(data);
}
```

Don't trace everything. A trace with 500 spans is as useless as no trace at all. Focus on service boundaries, external calls, and database queries. Internal method calls rarely need their own spans.

## Correlation IDs in Logs

Traces are great for visualizing request flow. But you also need to find the right logs. Spring Boot 3 puts the trace ID into the MDC automatically:

```yaml
logging:
  pattern:
    console: "%d{HH:mm:ss} [%X{traceId}] %-5level %logger{36} - %msg%n"
```

Now every log line from every service includes the trace ID. When a request fails, grab the trace ID from the error log, paste it into your trace UI, and see the entire request flow instantly.

This is the part that actually saves time in practice. Not the fancy trace visualizations - the ability to correlate logs across services with a single ID.

## The Stack

What's worked for us:
- **Micrometer Tracing** for instrumentation
- **OpenTelemetry** for the protocol
- **Grafana Tempo** for trace storage (it's free and integrates with Grafana)
- **Grafana** for visualization

Jaeger and Zipkin work too. The choice matters less than the fact that you have *something*. A basic tracing setup that exists is infinitely better than a perfect one that you'll set up "next sprint."

## The Real Value

Distributed tracing doesn't prevent bugs. It makes finding them fast. The difference between "the order service is slow" and "the order service's call to the inventory service's `/stock` endpoint takes 2 seconds because it's doing a full table scan" is the difference between a half-day investigation and a 5-minute fix.

Set it up early. You won't appreciate it until the first time something breaks in production and you can see exactly where and why in under a minute.
