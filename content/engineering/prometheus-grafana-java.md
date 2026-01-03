+++
title = "Prometheus and Grafana for Java Services, or Metrics That Actually Help"
date = 2026-01-03
[taxonomies]
tags = ["observability", "spring-boot", "devops"]
+++

You know what's worse than having no monitoring? Having monitoring that nobody looks at. Dashboards with twenty panels that all say "everything is fine" until everything isn't fine, and then they all turn red at once and you still don't know what happened.

I've been down this road. Setting up Prometheus and Grafana for Java services isn't hard. Setting them up so they actually help you debug production incidents - that takes more thought.

## Spring Boot Actuator: The Starting Point

If you're running Spring Boot and you haven't enabled Actuator metrics, you're flying blind. Two dependencies:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

Configuration:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, info, prometheus, metrics
  endpoint:
    health:
      show-details: when-authorized
  prometheus:
    metrics:
      export:
        enabled: true
```

Hit `/actuator/prometheus` and you get a wall of metrics in Prometheus exposition format. JVM memory, GC pauses, thread counts, HTTP request durations, connection pool stats - all out of the box.

Prometheus scrapes this endpoint at a configured interval (typically 15s or 30s), stores the time series, and you query it with PromQL or visualize it in Grafana.

## Micrometer Observation API

Micrometer is the metrics facade in Spring Boot. Think SLF4J but for metrics. It abstracts the metrics backend (Prometheus, Datadog, New Relic, etc.) so your code doesn't couple to a specific system.

The Observation API (Micrometer 1.10+) is the newer, higher-level abstraction. Instead of creating timers and counters separately, you create an "observation" that can emit metrics, traces, and logs:

```java
@Component
@RequiredArgsConstructor
public class OrderProcessor {

    private final ObservationRegistry observationRegistry;

    public void processOrder(Order order) {
        Observation.createNotStarted("order.processing", observationRegistry)
            .lowCardinalityKeyValue("order.type", order.getType().name())
            .lowCardinalityKeyValue("region", order.getRegion())
            .observe(() -> {
                // actual processing logic
                validateOrder(order);
                calculateTotal(order);
                saveOrder(order);
            });
    }
}
```

This creates a timer (`order.processing`), a counter (for successful/failed observations), and optionally a trace span. One API call, multiple signals.

The `lowCardinalityKeyValue` distinction matters. Low cardinality means a small number of distinct values (order type: STANDARD/EXPRESS/PRIORITY). High cardinality means many values (order ID: unique per order). Prometheus handles low cardinality well. High cardinality tags will blow up your metric cardinality and eat your storage.

## Custom Metrics

The built-in metrics cover the infrastructure. For business metrics, you write your own:

```java
@Component
@RequiredArgsConstructor
public class OrderMetrics {

    private final MeterRegistry registry;
    private final AtomicInteger activeOrders = new AtomicInteger(0);

    @PostConstruct
    void init() {
        Gauge.builder("orders.active", activeOrders, AtomicInteger::get)
            .description("Number of orders currently being processed")
            .register(registry);
    }

    public void orderCreated(Order order) {
        registry.counter("orders.created",
            "type", order.getType().name(),
            "region", order.getRegion()
        ).increment();
        activeOrders.incrementAndGet();
    }

    public void orderCompleted(Order order, Duration processingTime) {
        registry.timer("orders.processing.duration",
            "type", order.getType().name()
        ).record(processingTime);
        activeOrders.decrementAndGet();
    }

    public void orderFailed(Order order, Exception ex) {
        registry.counter("orders.failed",
            "type", order.getType().name(),
            "error", ex.getClass().getSimpleName()
        ).increment();
        activeOrders.decrementAndGet();
    }
}
```

Three types of metrics cover most needs:
- **Counter** - things that only go up: requests, errors, orders created
- **Gauge** - things that go up and down: active connections, queue depth, heap usage
- **Timer** - duration + count: request latency, processing time

Distribution summaries are for non-time measurements (request sizes, batch sizes), but timers cover 90% of my use cases.

## Grafana Dashboards

Here's where most setups fall apart. People create dashboards with every metric available and end up with thirty panels of noise.

My approach: one dashboard per service, four rows:

**Row 1: The Golden Signals**
- Request rate (PromQL: `rate(http_server_requests_seconds_count[5m])`)
- Error rate (PromQL: `rate(http_server_requests_seconds_count{status=~"5.."}[5m])`)
- Latency p50/p95/p99 (PromQL: `histogram_quantile(0.99, rate(http_server_requests_seconds_bucket[5m]))`)
- Saturation (active threads or connection pool usage)

**Row 2: JVM Health**
- Heap usage (used vs committed vs max)
- GC pause duration and frequency
- Thread count
- Non-heap memory (Metaspace)

**Row 3: Dependencies**
- Database connection pool (active, idle, pending)
- HTTP client latency to downstream services
- Kafka consumer lag (if applicable)
- Cache hit rates (if applicable)

**Row 4: Business Metrics**
- Orders per minute (or whatever your service does)
- Error breakdown by type
- Processing duration by category

Four rows. Twelve to sixteen panels. That's it. If you need more detail, create a separate deep-dive dashboard for specific subsystems.

## Spring Boot Admin

For teams that want a simpler alternative to Grafana, Spring Boot Admin provides a web UI for monitoring Spring Boot applications. It uses Actuator endpoints and shows health, metrics, environment, configuration, and even log levels that you can change at runtime.

```xml
<!-- Admin Server -->
<dependency>
    <groupId>de.codecentric</groupId>
    <artifactId>spring-boot-admin-starter-server</artifactId>
</dependency>

<!-- In each client application -->
<dependency>
    <groupId>de.codecentric</groupId>
    <artifactId>spring-boot-admin-starter-client</artifactId>
</dependency>
```

It's useful for development and small deployments. For production at scale, Prometheus + Grafana is the better choice because it handles high cardinality, long-term storage, and alerting properly.

## JMX Monitoring

JMX still exists and is still useful. Micrometer exports metrics to JMX by default. For production, you're unlikely to use JMX directly (Prometheus is better), but for local debugging, JConsole or VisualVM connected via JMX gives you real-time heap analysis, thread dumps, and MBean access.

The more practical use of JMX in production: Prometheus JMX Exporter. If you have legacy Java services that don't use Micrometer, the JMX Exporter agent runs alongside your JVM and exposes JMX MBeans as Prometheus metrics:

```yaml
# Run as a Java agent
java -javaagent:jmx_prometheus_javaagent.jar=9404:config.yaml -jar myapp.jar
```

This is the bridge for services where adding Micrometer isn't an option. The JMX Exporter translates HikariCP, Tomcat, and JVM MBeans into Prometheus metrics without code changes.

## Alerting: The Part People Skip

Dashboards are useless at 3 AM if nobody is looking at them. Alerting is where monitoring becomes useful.

Prometheus alerting rules:

```yaml
groups:
  - name: order-service
    rules:
      - alert: HighErrorRate
        expr: rate(http_server_requests_seconds_count{application="order-service", status=~"5.."}[5m]) / rate(http_server_requests_seconds_count{application="order-service"}[5m]) > 0.05
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Order service error rate above 5%"

      - alert: HighLatency
        expr: histogram_quantile(0.99, rate(http_server_requests_seconds_bucket{application="order-service"}[5m])) > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Order service p99 latency above 2 seconds"

      - alert: ConnectionPoolExhaustion
        expr: hikaricp_connections_active{application="order-service"} / hikaricp_connections_max{application="order-service"} > 0.9
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Connection pool above 90% utilization"
```

Three alerts. Error rate, latency, and connection pool. These cover the most common failure modes I've encountered. Add more as you learn what breaks in your specific system.

The `for` clause is important. It prevents alerting on transient spikes. A brief latency spike during a GC pause isn't worth waking someone up. A sustained high error rate for 2 minutes is.

Metrics without alerting is a hobby. Metrics with alerting is operations. Know the difference.
