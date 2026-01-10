+++
title = "Optimistic vs Pessimistic Locking: When to Use Each"
date = 2025-08-15
[taxonomies]
tags = ["database", "jpa", "concurrency"]
+++

Locking is one of those topics where the theory is simple and the practice is full of surprises. I've shipped bugs with both optimistic and pessimistic locking, and each time I learned something I should have already known.

## Optimistic Locking: Hope for the Best

Optimistic locking assumes conflicts are rare. You read data, do your work, and at write time, check whether anyone else modified the data while you were working. If they did, you fail and retry.

In JPA, it's a `@Version` column:

```java
@Entity
public class Order {
    @Id
    private String id;

    @Version
    private Long version;

    private OrderStatus status;
    private BigDecimal total;
}
```

When Hibernate updates this entity, it includes the version in the WHERE clause:

```sql
UPDATE orders SET status = 'SHIPPED', version = 4
WHERE id = 'order-123' AND version = 3;
```

If the version doesn't match (because another transaction incremented it), the update affects 0 rows and Hibernate throws `OptimisticLockException`. No database locks are held during the transaction. The conflict is detected at commit time.

### When It Works

Optimistic locking works beautifully when:

- Conflicts are rare (most use cases)
- The operation can be retried without side effects
- You want maximum read concurrency

For a typical CRUD application where users edit different records most of the time, optimistic locking is the right default. Conflicts happen occasionally, and the retry cost is low.

### When It Doesn't

Optimistic locking falls apart when conflicts are frequent. If 10 transactions are all trying to update the same row simultaneously, 9 of them will fail and retry. Those retries might also fail. You end up with a retry storm that's worse than just serializing the updates.

I've seen this with inventory counters, seat reservations, and any "decrement a shared counter" pattern. Optimistic locking is the wrong tool when contention is high.

### Handling the Exception

```java
@Retryable(
    retryFor = OptimisticLockingFailureException.class,
    maxAttempts = 3,
    backoff = @Backoff(delay = 50, multiplier = 2)
)
@Transactional
public void updateOrderStatus(String orderId, OrderStatus newStatus) {
    Order order = orderRepository.findById(orderId)
        .orElseThrow(() -> new NotFoundException(orderId));
    order.setStatus(newStatus);
}
```

Spring Retry handles the retry loop. The exponential backoff gives the competing transaction time to finish. Three attempts is usually enough: if you're still conflicting after three retries, something structural is wrong.

## Pessimistic Locking: Lock It Down

Pessimistic locking acquires a database lock when reading the data, preventing other transactions from modifying it until you're done.

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT o FROM Order o WHERE o.id = :id")
Optional<Order> findByIdForUpdate(@Param("id") String id);
```

This generates `SELECT ... FOR UPDATE`, which acquires a row-level exclusive lock. Other transactions trying to read this row with `FOR UPDATE` (or modify it) will block until your transaction commits or rolls back.

### When It Works

Pessimistic locking works when:

- Contention is high (many transactions competing for the same rows)
- You need to guarantee that your read-modify-write cycle is atomic
- The transaction is short-lived (you don't want to hold locks for long)

```java
@Transactional
public void decrementInventory(String productId, int quantity) {
    Product product = productRepository.findByIdForUpdate(productId);
    if (product.getStock() < quantity) {
        throw new InsufficientStockException();
    }
    product.setStock(product.getStock() - quantity);
}
```

The `FOR UPDATE` lock ensures that between reading the stock and writing the new value, nobody else can change it. No retries needed. The lock serializes concurrent access.

### Lock Timeout

Don't let transactions wait forever for a lock:

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@QueryHints(@QueryHint(name = "jakarta.persistence.lock.timeout", value = "3000"))
@Query("SELECT o FROM Order o WHERE o.id = :id")
Optional<Order> findByIdForUpdate(@Param("id") String id);
```

A 3-second timeout means the query throws `LockTimeoutException` if it can't acquire the lock in time. Handle it:

```java
try {
    productRepository.findByIdForUpdate(productId);
} catch (LockTimeoutException e) {
    throw new ServiceUnavailableException("Resource is busy, try again later");
}
```

## Advisory Locks: Application-Level Locks

PostgreSQL supports advisory locks: locks that aren't tied to any table or row. They're application-level locks managed by the database.

```java
@Query(value = "SELECT pg_try_advisory_lock(:lockId)", nativeQuery = true)
boolean tryAcquireLock(@Param("lockId") long lockId);

@Query(value = "SELECT pg_advisory_unlock(:lockId)", nativeQuery = true)
boolean releaseLock(@Param("lockId") long lockId);
```

Use cases:
- Preventing duplicate cron job execution across multiple instances
- Locking a business process (e.g., "only one thread can process customer X's batch at a time")
- Distributed mutex without external infrastructure (Redis, Zookeeper)

Advisory locks are lightweight and don't conflict with regular row locks. The downside: they're PostgreSQL-specific and if your code crashes without releasing them, session-level locks persist until the connection closes (transaction-level locks auto-release on commit/rollback).

I use `pg_try_advisory_xact_lock` for transaction-scoped locks: they release automatically when the transaction ends, which eliminates the "forgot to unlock" class of bugs.

## SKIP LOCKED: Job Queues Without External Infrastructure

`SKIP LOCKED` is a game-changer for simple job queues:

```sql
SELECT * FROM jobs
WHERE status = 'PENDING'
ORDER BY created_at
LIMIT 1
FOR UPDATE SKIP LOCKED;
```

This selects the first pending job, locking it. If another worker already locked a pending job, `SKIP LOCKED` skips over it instead of waiting. Multiple workers can pull jobs from the same table without blocking each other.

In Spring Data:

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@QueryHints(@QueryHint(name = "jakarta.persistence.lock.timeout", value = "-2")) // SKIP LOCKED
@Query("SELECT j FROM Job j WHERE j.status = 'PENDING' ORDER BY j.createdAt")
List<Job> findNextPendingJobs(Pageable pageable);
```

Note: the `lock.timeout = -2` hint for SKIP LOCKED is Hibernate-specific. Check your Hibernate version's documentation.

I've used this pattern to avoid deploying RabbitMQ or Redis for simple background job processing. It works surprisingly well for moderate throughput (hundreds of jobs per second). For higher throughput, use a proper message broker.

## When to Use Each

| Scenario | Strategy |
|---|---|
| General CRUD, low contention | Optimistic (`@Version`) |
| High-contention counters (inventory, seats) | Pessimistic (`FOR UPDATE`) |
| Read-heavy, rare writes | Optimistic |
| Short-lived critical sections | Pessimistic |
| Job queue without external broker | Pessimistic with `SKIP LOCKED` |
| Distributed process coordination | Advisory locks |
| Long-running user edits (edit form) | Optimistic |

## My Default

Start with optimistic locking. It's simpler, doesn't hold database locks, and works for the vast majority of use cases. Add `@Version` to every entity. Implement retry logic for `OptimisticLockingFailureException`.

Switch to pessimistic locking only when you have evidence of high contention: meaning your optimistic locking retries are failing repeatedly. And when you do switch, scope the lock as narrowly as possible. Lock one row, not a table. Keep the transaction short. Release the lock as quickly as possible.

The worst thing you can do is pessimistically lock everything "just to be safe." That's how you turn a concurrent system into a sequential one.
