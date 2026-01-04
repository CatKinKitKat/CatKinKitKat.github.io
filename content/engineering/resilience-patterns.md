+++
title = "Resilience Patterns That Actually Work in Spring Boot"
date = 2026-01-04
[taxonomies]
tags = ["microservices", "spring-boot", "patterns"]
+++

The first time a downstream service went down and took our entire platform with it, I learned a valuable lesson: hope is not a resilience strategy. Neither is "it'll probably be fine."

If you're building microservices (or even a monolith that calls external APIs), you need resilience patterns. Not because they're fashionable - because the alternative is getting paged at 2 AM when a third-party payment gateway decides to respond with 30-second timeouts instead of errors.

Resilience4j is the library I reach for. It's lightweight, modular, and integrates cleanly with Spring Boot. Here's what I actually use in production.

## Circuit Breaker

The circuit breaker is the most important pattern here. When a downstream service is failing, stop calling it. Give it time to recover instead of hammering it with requests that are all going to fail anyway.

```yaml
resilience4j:
  circuitbreaker:
    instances:
      paymentService:
        slidingWindowSize: 10
        failureRateThreshold: 50
        waitDurationInOpenState: 10s
        permittedNumberOfCallsInHalfOpenState: 3
        slidingWindowType: COUNT_BASED
```

```java
@CircuitBreaker(name = "paymentService", fallbackMethod = "paymentFallback")
public PaymentResult processPayment(PaymentRequest request) {
    return paymentClient.charge(request);
}

private PaymentResult paymentFallback(PaymentRequest request, Exception ex) {
    // Queue for retry, return pending status, whatever makes sense
    log.warn("Payment service unavailable, queuing for retry: {}", ex.getMessage());
    retryQueue.enqueue(request);
    return PaymentResult.pending(request.getOrderId());
}
```

The sliding window tracks the last N calls. If more than 50% fail, the circuit opens and all calls go straight to the fallback for 10 seconds. After that, it lets 3 calls through to test if the service is back. Simple, effective.

The key decision is your fallback strategy. For some services, returning a cached result is fine. For others, queuing for later retry makes sense. For critical paths where there's no acceptable fallback... well, that's a design problem you should have caught earlier.

## Retry

Not every failure is permanent. Network blips, brief GC pauses on the downstream service, transient database locks - these resolve themselves if you just try again.

```yaml
resilience4j:
  retry:
    instances:
      inventoryService:
        maxAttempts: 3
        waitDuration: 500ms
        enableExponentialBackoff: true
        exponentialBackoffMultiplier: 2
        retryExceptions:
          - java.io.IOException
          - java.net.SocketTimeoutException
        ignoreExceptions:
          - com.example.BusinessValidationException
```

```java
@Retry(name = "inventoryService")
@CircuitBreaker(name = "inventoryService", fallbackMethod = "inventoryFallback")
public StockLevel checkStock(String productId) {
    return inventoryClient.getStockLevel(productId);
}
```

Two things that matter here. First, exponential backoff. If the first call failed, don't immediately retry - give the downstream service a moment. 500ms, then 1s, then 2s. Second, the exception filtering. Retry on IOExceptions (transient network issues), don't retry on business validation errors (retrying a bad request won't make it valid).

Order matters when combining annotations. The retry wraps the circuit breaker, so all retry attempts happen before the circuit breaker counts a failure. That's usually what you want.

## Rate Limiting

Sometimes you're the problem. If your service can generate bursts of traffic that overwhelm a downstream dependency, rate limiting protects everyone.

```yaml
resilience4j:
  ratelimiter:
    instances:
      externalApi:
        limitForPeriod: 100
        limitRefreshPeriod: 1s
        timeoutDuration: 500ms
```

```java
@RateLimiter(name = "externalApi")
public ExchangeRate getExchangeRate(String currency) {
    return exchangeRateClient.getRate(currency);
}
```

This caps calls to 100 per second. If you exceed that, the call blocks for up to 500ms waiting for a permit. After that, it throws a `RequestNotPermitted` exception.

I use this primarily for external APIs with rate limits (payment gateways, third-party data providers) and for internal services that I know can't handle unlimited load. It's also useful for protecting yourself during retry storms - if a downstream service recovers and every client retries simultaneously, rate limiting prevents the thundering herd.

## Bulkhead

The bulkhead pattern isolates failures. If your service calls five downstream services and one is slow, you don't want that one slow service to consume all your threads and starve the other four.

```yaml
resilience4j:
  bulkhead:
    instances:
      reportService:
        maxConcurrentCalls: 10
        maxWaitDuration: 100ms
  thread-pool-bulkhead:
    instances:
      analyticsService:
        maxThreadPoolSize: 5
        coreThreadPoolSize: 3
        queueCapacity: 10
```

```java
@Bulkhead(name = "reportService")
public Report generateReport(ReportRequest request) {
    return reportClient.generate(request);
}
```

Two flavors here. The semaphore bulkhead limits concurrent calls on the calling thread - simple and lightweight. The thread pool bulkhead runs calls on a separate thread pool - more isolation but with the overhead of thread context switching.

I default to semaphore bulkheads unless I need the stronger isolation. With virtual threads in modern Java, the thread pool bulkhead is less relevant anyway.

## Timeout / Time Limiting

This is the one people forget. You set up circuit breakers and retries but don't set timeouts, so a slow downstream service ties up your threads for the default 30-second HTTP client timeout.

```yaml
resilience4j:
  timelimiter:
    instances:
      searchService:
        timeoutDuration: 2s
        cancelRunningFuture: true
```

```java
@TimeLimiter(name = "searchService")
@CircuitBreaker(name = "searchService", fallbackMethod = "searchFallback")
public CompletableFuture<SearchResults> search(String query) {
    return CompletableFuture.supplyAsync(() -> searchClient.search(query));
}
```

Two seconds. If the search service can't respond in two seconds, we cancel and fall back. This is aggressive, and that's intentional. Your users aren't going to wait 30 seconds for a search result. Fail fast and give them a degraded experience rather than a frozen page.

Set your timeouts based on your SLA, not on how long the downstream service *might* take. If your page needs to load in 3 seconds, and you have 4 downstream calls, each one gets well under a second.

## Combining Everything

In practice, I stack these patterns. Here's the order that makes sense:

```java
@Retry(name = "paymentService")
@CircuitBreaker(name = "paymentService", fallbackMethod = "fallback")
@RateLimiter(name = "paymentService")
@Bulkhead(name = "paymentService")
@TimeLimiter(name = "paymentService")
public CompletableFuture<PaymentResult> processPayment(PaymentRequest req) {
    return CompletableFuture.supplyAsync(() -> paymentClient.charge(req));
}
```

The execution order (innermost to outermost): TimeLimiter, Bulkhead, RateLimiter, CircuitBreaker, Retry. So a call first checks the time limit, then the bulkhead, then rate limiting, then the circuit breaker decides whether to let it through, and retries wrap the whole thing.

## The Actuator Integration

Don't skip monitoring. Resilience4j exposes metrics through Spring Boot Actuator, which feeds into Prometheus/Grafana.

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,circuitbreakers,ratelimiters,retries
  health:
    circuitbreakers:
      enabled: true
```

You want to see your circuit breaker states on a dashboard. When something is in the open state, that's a signal that needs investigation, not just a metric to glance at.

## What I've Learned the Hard Way

Resilience patterns are not a substitute for fixing the root cause. If your downstream service is flaky, the circuit breaker buys you time, but you still need to fix the service. I've seen teams get comfortable with their fallbacks and never address why the payment service goes down every Thursday at 4 PM.

Also, test your fallbacks. They're code paths that rarely execute, which means they're code paths that are rarely correct. Run chaos engineering exercises - kill a downstream service intentionally and verify that the fallback actually works. The worst time to discover your fallback throws a NullPointerException is during a real outage.
