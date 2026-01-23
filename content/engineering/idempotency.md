+++
title = "Idempotency in Distributed Systems, or Why Your Payment Charged Twice"
date = 2026-01-23
[taxonomies]
tags = ["distributed-systems", "patterns", "microservices"]
+++

A user clicks "Pay Now." The request times out. They click again. Congratulations, you've charged them twice. This is the kind of bug that gets you a meeting with the CTO, and it's entirely preventable.

Idempotency means that performing the same operation multiple times produces the same result as performing it once. In distributed systems, where network failures, retries, and duplicate messages are facts of life, idempotency isn't optional. It's a survival requirement.

## The Problem

Networks are unreliable. HTTP requests can fail after the server has processed them but before the client receives the response. Message queues deliver at-least-once, not exactly-once (despite what some marketing pages claim). Load balancers retry failed requests. Clients retry on timeout.

All of these scenarios produce duplicate requests. If your system isn't idempotent, duplicates cause real damage: double charges, duplicate orders, inconsistent inventory counts.

## Naturally Idempotent Operations

Some operations are idempotent by nature. PUT and DELETE in HTTP are designed to be idempotent. Setting a value is idempotent, `setBalance(100)` produces the same result whether you call it once or ten times. Reading data is idempotent.

But most business operations aren't naturally idempotent. `addToBalance(50)` called twice gives a different result than calling it once. `createOrder()` called twice creates two orders. These need explicit idempotency handling.

## Idempotency Keys

The most common pattern: the client generates a unique key for each logical operation and includes it in the request. The server uses this key to detect and deduplicate retries.

```java
@PostMapping("/api/payments")
public ResponseEntity<PaymentResult> processPayment(
        @RequestHeader("Idempotency-Key") String idempotencyKey,
        @RequestBody PaymentRequest request) {

    // Check if we've already processed this key
    Optional<PaymentResult> existing = idempotencyStore.find(idempotencyKey);
    if (existing.isPresent()) {
        return ResponseEntity.ok(existing.get());
    }

    // Process the payment
    PaymentResult result = paymentService.charge(request);

    // Store the result keyed by the idempotency key
    idempotencyStore.save(idempotencyKey, result);

    return ResponseEntity.ok(result);
}
```

The critical detail: the idempotency check and the business operation must be atomic. If you check the key, process the payment, and then store the key as three separate steps, there's a window where a duplicate request slips through between the check and the store.

```java
@Transactional
public PaymentResult processPaymentIdempotently(String idempotencyKey, PaymentRequest request) {
    // Acquire a lock on the idempotency key (prevents concurrent duplicates)
    if (idempotencyStore.tryLock(idempotencyKey)) {
        Optional<PaymentResult> existing = idempotencyStore.find(idempotencyKey);
        if (existing.isPresent()) {
            return existing.get();
        }

        PaymentResult result = paymentService.charge(request);
        idempotencyStore.save(idempotencyKey, result);
        return result;
    }

    throw new ConcurrentRequestException("Duplicate request in progress");
}
```

## The Deduplication Table

For message consumers (Kafka, RabbitMQ, etc.), a deduplication table serves the same purpose as idempotency keys.

```sql
CREATE TABLE processed_messages (
    message_id VARCHAR(255) PRIMARY KEY,
    processed_at TIMESTAMP DEFAULT NOW(),
    result JSONB
);
```

```java
@KafkaListener(topics = "payment-events")
@Transactional
public void handlePaymentEvent(PaymentEvent event) {
    // Check if already processed
    if (processedMessageRepository.existsById(event.getMessageId())) {
        log.info("Duplicate message {}, skipping", event.getMessageId());
        return;
    }

    // Process the event
    orderService.updatePaymentStatus(event);

    // Mark as processed (same transaction as the business logic)
    processedMessageRepository.save(
        new ProcessedMessage(event.getMessageId())
    );
}
```

The table grows over time. You'll need a cleanup strategy (TTL-based deletion, partitioning by date, or moving to a time-windowed approach where you only check for duplicates within a reasonable window: an hour, a day, whatever matches your retry semantics).

## The Synchrony Budget

Here's a concept I find useful but rarely see discussed. Every distributed system has a "synchrony budget": the amount of time during which the system needs to behave synchronously to maintain correctness.

For idempotency, this means: how long do you need to remember that you've processed a request? If your client retries within 5 minutes, your idempotency window needs to be at least 5 minutes. If your message queue can redeliver messages up to 24 hours later, your deduplication window needs to be at least 24 hours.

```java
@Scheduled(cron = "0 0 3 * * *") // 3 AM daily
public void cleanupIdempotencyKeys() {
    Instant cutoff = Instant.now().minus(Duration.ofDays(7));
    idempotencyStore.deleteOlderThan(cutoff);
    log.info("Cleaned up idempotency keys older than {}", cutoff);
}
```

The synchrony budget also affects your design choices. A short budget (seconds to minutes) can be handled with a Redis cache. A long budget (hours to days) needs a durable store like PostgreSQL. An infinite budget (you can never process duplicates) means you need the deduplication check to be permanent, or you need to make the operation naturally idempotent.

## Making Operations Naturally Idempotent

Sometimes you can redesign the operation to be naturally idempotent, which is cleaner than maintaining deduplication tables.

```java
// NOT idempotent: incrementing a counter
UPDATE accounts SET balance = balance + 50 WHERE id = ?;

// Idempotent: setting to a specific state with a version check
UPDATE accounts SET balance = 150, version = 3
WHERE id = ? AND version = 2;

// Idempotent: using the operation ID as a unique constraint
INSERT INTO ledger_entries (operation_id, account_id, amount)
VALUES (?, ?, ?)
ON CONFLICT (operation_id) DO NOTHING;
```

The ledger approach is my favorite for financial operations. Each credit or debit gets a unique operation ID. The unique constraint on operation_id means you literally cannot insert the same entry twice. The balance is derived from the sum of ledger entries, so it's always correct regardless of how many times you retry.

## CAP Theorem Implications

Idempotency interacts with the CAP theorem in interesting ways. Your deduplication store is itself a distributed component. What happens when it's unavailable?

If you prioritize consistency (CP): reject requests when the deduplication store is unreachable. No duplicate processing, but reduced availability.

If you prioritize availability (AP): process requests even when you can't check for duplicates. Higher availability, but risk of duplicate processing.

For payment systems, I always choose CP. A temporary refusal is better than a double charge. For less critical systems (sending notifications, updating analytics), AP might be acceptable: a duplicate notification is annoying but not harmful.

```java
public PaymentResult processPayment(String idempotencyKey, PaymentRequest request) {
    try {
        return processPaymentIdempotently(idempotencyKey, request);
    } catch (DeduplicationStoreUnavailableException e) {
        // CP choice: fail rather than risk duplicate processing
        log.error("Deduplication store unavailable, rejecting request");
        throw new ServiceUnavailableException("Please retry later");
    }
}
```

## Idempotency at the API Level

Stripe gets this right. Every mutation endpoint accepts an `Idempotency-Key` header. If you send the same key twice, you get the same response. The key is scoped to your API key (so different clients can use the same key independently), and it expires after 24 hours.

If you're building an API that accepts payments, creates resources, or does anything with side effects, steal Stripe's approach. It's well-documented, widely understood, and battle-tested.

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Idempotent {
    long ttlMinutes() default 1440; // 24 hours
}

@Aspect
@Component
public class IdempotencyAspect {

    @Around("@annotation(idempotent)")
    public Object enforceIdempotency(ProceedingJoinPoint joinPoint, Idempotent idempotent) throws Throwable {
        String key = extractIdempotencyKey();
        if (key == null) {
            throw new BadRequestException("Idempotency-Key header required");
        }

        return idempotencyService.executeIdempotently(
            key,
            Duration.ofMinutes(idempotent.ttlMinutes()),
            () -> {
                try { return joinPoint.proceed(); }
                catch (Throwable t) { throw new RuntimeException(t); }
            }
        );
    }
}
```

## The Bottom Line

In a distributed system, duplicate requests are not a possibility; they're a certainty. Design for them from the start, not as an afterthought when a customer reports they were charged three times.

Idempotency keys for APIs. Deduplication tables for message consumers. Natural idempotency where you can achieve it through design. And when in doubt, choose consistency over availability for operations with real-world consequences.

The extra code is boring. The extra table is boring. But "boring" beats "explaining to the finance team why 200 customers got double-charged" every single time.
