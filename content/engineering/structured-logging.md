+++
title = "Structured Logging, or How to Make Logs Actually Searchable"
date = 2026-01-06
[taxonomies]
tags = ["observability", "spring-boot", "devops"]
+++

Logs are the oldest observability signal. They're also the most abused. I've seen logs that look like this:

```
2024-01-15 10:23:45 INFO OrderService - Processing order for customer John
2024-01-15 10:23:45 ERROR OrderService - Something went wrong!
```

What order? What customer ID? What went wrong? When you have one service and ten users, you can grep through this. When you have fifty services processing thousands of requests per second, unstructured logs are noise.

Structured logging means emitting logs as key-value pairs (usually JSON) instead of human-formatted strings. It turns logs from text files into queryable data.

## JSON Logging in Spring Boot

Spring Boot uses Logback by default. Structured JSON output is a configuration change, not a code change.

In Spring Boot 3.4+, structured logging is built in:

```yaml
logging:
  structured:
    format:
      console: ecs    # or logstash, gelf
```

For earlier versions, use Logstash Logback Encoder:

```xml
<dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
    <version>7.4</version>
</dependency>
```

In `logback-spring.xml`:

```xml
<configuration>
    <appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <includeMdcKeyName>correlationId</includeMdcKeyName>
            <includeMdcKeyName>userId</includeMdcKeyName>
        </encoder>
    </appender>

    <root level="INFO">
        <appender-ref ref="JSON"/>
    </root>
</configuration>
```

Your logs now look like:

```json
{
  "@timestamp": "2024-01-15T10:23:45.123Z",
  "level": "INFO",
  "logger_name": "com.example.OrderService",
  "message": "Processing order",
  "orderId": "ORD-789",
  "customerId": "CUST-456",
  "correlationId": "abc-def-123",
  "thread_name": "http-nio-8080-exec-1",
  "application": "order-service"
}
```

Every field is searchable. Want all logs for order ORD-789? Query `orderId:ORD-789`. Want all errors for customer CUST-456? Query `customerId:CUST-456 AND level:ERROR`. This is the whole point.

## MDC: Logging Context

MDC (Mapped Diagnostic Context) is how you attach contextual information to every log line in a request without passing it through every method signature.

```java
@Component
public class CorrelationIdFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                     HttpServletResponse response,
                                     FilterChain filterChain) throws ServletException, IOException {
        String correlationId = request.getHeader("X-Correlation-ID");
        if (correlationId == null) {
            correlationId = UUID.randomUUID().toString();
        }

        MDC.put("correlationId", correlationId);
        MDC.put("userId", extractUserId(request));

        try {
            response.setHeader("X-Correlation-ID", correlationId);
            filterChain.doFilter(request, response);
        } finally {
            MDC.clear();  // critical - prevent context leaking to the next request
        }
    }
}
```

Once the MDC is set, every log statement in that request's execution automatically includes the correlation ID and user ID. Your service code doesn't need to know about it:

```java
// This is all you write
log.info("Processing order {}", orderId);

// The JSON output includes correlationId, userId, etc. from MDC
```

With virtual threads, be careful: MDC uses `ThreadLocal` by default. Virtual threads may run on different carrier threads. Micrometer's context propagation (`context-propagation` library) handles this correctly for Spring Boot 3.2+. Make sure you have it on the classpath.

## Logging Levels and Strategy

Every team argues about this. Here's what I've settled on:

- **ERROR** - something is broken and needs attention. An unhandled exception, a failed payment, a circuit breaker opening. If it fires, someone should investigate.
- **WARN** - something is wrong but the system handled it. A retry succeeded, a fallback was used, a validation failed. Worth monitoring in aggregate.
- **INFO** - normal operations that matter. Request received, order processed, message consumed. One or two lines per request, not twenty.
- **DEBUG** - development-time detail. Method entry/exit, intermediate state, full request/response bodies. Off in production by default.
- **TRACE** - framework-level detail. Almost never useful outside of debugging framework internals.

The most common mistake: logging at INFO level inside loops or hot paths. If you process 1000 items in a batch and log each one at INFO, that's 1000 log lines per batch. Log the batch start and end at INFO, individual items at DEBUG.

## Profile-Specific Logging

Different environments need different logging configurations:

```yaml
# application.yml (default)
logging:
  level:
    root: INFO
    com.example: INFO
    org.hibernate.SQL: WARN

---
# application-dev.yml
logging:
  level:
    com.example: DEBUG
    org.hibernate.SQL: DEBUG
    org.hibernate.orm.jdbc.bind: TRACE
  pattern:
    console: "%d{HH:mm:ss.SSS} %-5level [%thread] %logger{36} - %msg%n"

---
# application-prod.yml
logging:
  structured:
    format:
      console: ecs
  level:
    root: INFO
    com.example: INFO
```

Development gets human-readable logs with debug detail. Production gets structured JSON at INFO level. This prevents the "I turned on debug logging in production and the disk filled up" scenario.

## Hibernate and SQL Query Logging

SQL query logging is a special case because it's simultaneously essential for debugging and catastrophic for performance if left on.

```yaml
# Show SQL (formatted)
spring:
  jpa:
    show-sql: false  # never use this - it bypasses the logging framework
    properties:
      hibernate:
        format_sql: true

logging:
  level:
    org.hibernate.SQL: DEBUG          # logs the query
    org.hibernate.orm.jdbc.bind: TRACE  # logs the bind parameters
```

`spring.jpa.show-sql=true` writes directly to stdout, bypassing Logback entirely. Your log aggregation system won't capture it, and it won't be structured. Always use the `logging.level` approach instead.

For production, keep these at WARN. For debugging an N+1 query problem, turn them on temporarily. Spring Boot Admin lets you change log levels at runtime without redeploying - that's one of its most useful features.

## Correlation IDs

In a microservices architecture, a single user request fans out to multiple services. Without correlation IDs, correlating logs across services is impossible.

The pattern:
1. The API gateway (or first service) generates a correlation ID
2. It's passed to downstream services via HTTP headers (`X-Correlation-ID`) or message headers
3. Each service adds it to the MDC
4. Every log line includes the correlation ID

With Spring Cloud Sleuth (or Micrometer Tracing in Spring Boot 3+), this happens automatically. The trace ID serves as the correlation ID:

```yaml
management:
  tracing:
    sampling:
      probability: 1.0  # sample everything (adjust for production)
```

Micrometer Tracing propagates the trace ID across HTTP calls (via W3C Trace Context headers) and Kafka messages. Your logs automatically include `traceId` and `spanId`.

Query your log aggregation system with `traceId:abc123` and you get every log line from every service involved in that request. This is the single most useful thing in distributed systems debugging.

## ELK vs Loki: Where Do the Logs Go?

You need somewhere to send these structured logs. Two main options:

**ELK (Elasticsearch + Logstash + Kibana)** - the established choice. Full-text search, complex queries, powerful visualizations. Also resource-hungry and expensive to operate at scale.

**Loki (Grafana Loki)** - the newer alternative. Doesn't index log content, only labels. Much cheaper to run. Queries are less powerful but adequate for most use cases. Integrates naturally with Grafana (which you're probably already using for metrics).

I've used both. Loki is my default recommendation for teams that don't need complex log analytics. ELK is the choice when you're doing serious log analysis, compliance, or security auditing.

But that's a whole separate article.

## The Non-Negotiables

Regardless of your stack, these are the things every service should do:

1. **Structured output.** JSON in production. Human-readable in development.
2. **Correlation IDs.** On every log line. Propagated across services.
3. **Contextual information.** User ID, order ID, request ID in the MDC.
4. **No sensitive data.** Passwords, tokens, PII - never in logs. Use a custom Logback filter to mask sensitive fields if needed.
5. **Consistent field names.** If one service logs `userId` and another logs `user_id`, your queries break. Agree on naming conventions.

Logging seems simple. It's not. But getting it right makes every production incident significantly less painful. And that's really the whole point.
