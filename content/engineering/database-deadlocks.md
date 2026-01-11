+++
title = "Database Deadlocks - Causes, Prevention, and That One Time at 2 AM"
date = 2025-09-01
[taxonomies]
tags = ["database", "debugging"]
+++

The first time I got paged for a deadlock in production, I didn't know what I was looking at. The service was throwing `CannotAcquireLockException` intermittently, and the database logs were full of "deadlock detected" messages. It took me four hours to understand the cause. Now I can spot deadlock-prone code in a code review. Here's everything I learned the hard way.

## What a Deadlock Is

Two (or more) transactions are each waiting for a lock held by the other. Neither can proceed. The database detects this and kills one of them (the "victim") so the other can complete.

```
Transaction A: locks row 1, wants to lock row 2
Transaction B: locks row 2, wants to lock row 1
-- Deadlock. Database aborts one transaction.
```

The database doesn't prevent deadlocks from occurring. It detects them after the fact and resolves them by aborting the transaction with the least work invested (usually). Your application receives an exception and has to deal with it.

## Common Java/JPA Patterns That Cause Deadlocks

### Pattern 1: Inconsistent Lock Ordering

This is the most common cause. Two code paths lock the same rows but in different orders.

```java
// Service A: transfer money
@Transactional
public void transfer(String fromId, String toId, BigDecimal amount) {
    Account from = accountRepository.findByIdForUpdate(fromId); // Lock from
    Account to = accountRepository.findByIdForUpdate(toId);     // Lock to
    from.debit(amount);
    to.credit(amount);
}
```

If two concurrent transfers happen - one from A to B and one from B to A - you get a deadlock. Transaction 1 locks A and waits for B. Transaction 2 locks B and waits for A.

The fix: always lock rows in a consistent order.

```java
@Transactional
public void transfer(String fromId, String toId, BigDecimal amount) {
    String first = fromId.compareTo(toId) < 0 ? fromId : toId;
    String second = fromId.compareTo(toId) < 0 ? toId : fromId;

    Account firstAccount = accountRepository.findByIdForUpdate(first);
    Account secondAccount = accountRepository.findByIdForUpdate(second);

    // Now debit/credit based on which is from/to
    if (first.equals(fromId)) {
        firstAccount.debit(amount);
        secondAccount.credit(amount);
    } else {
        secondAccount.debit(amount);
        firstAccount.credit(amount);
    }
}
```

It's ugly, but it's deadlock-free. Both transactions always lock the lower ID first.

### Pattern 2: Bulk Updates Without Ordering

```java
@Transactional
public void archiveOrders(List<String> orderIds) {
    for (String id : orderIds) {
        Order order = orderRepository.findByIdForUpdate(id);
        order.setStatus(OrderStatus.ARCHIVED);
    }
}
```

If two calls to `archiveOrders` have overlapping but differently ordered ID lists, deadlock. The fix: sort the IDs before processing.

```java
List<String> sorted = orderIds.stream().sorted().toList();
for (String id : sorted) { ... }
```

### Pattern 3: Foreign Key Cascades

This one is insidious. Inserting a child row acquires a shared lock on the parent row (to verify the foreign key constraint). If another transaction is updating the parent row while you're inserting a child, you can deadlock.

```
T1: UPDATE orders SET status = 'SHIPPED' WHERE id = 1;  - Exclusive lock on order 1
T2: INSERT INTO line_items (order_id, ...) VALUES (1, ...);  - Needs shared lock on order 1
T2: Waits for T1's exclusive lock...
T1: INSERT INTO line_items (order_id, ...) VALUES (2, ...);  - Needs shared lock on order 2
    - But T2 might hold a lock on order 2 from an earlier statement
```

The fix depends on the situation. Sometimes reordering operations helps. Sometimes you need to explicitly lock the parent row before inserting children.

### Pattern 4: Index Gap Locks (MySQL/InnoDB)

InnoDB uses gap locks in REPEATABLE READ isolation. When you do a range scan with `FOR UPDATE`, InnoDB locks the gaps between index entries, not just the existing rows. Two transactions inserting into adjacent gaps can deadlock.

```sql
T1: SELECT * FROM orders WHERE status = 'PENDING' FOR UPDATE;  - Locks gaps
T2: INSERT INTO orders (status) VALUES ('PENDING');  - Waits for gap lock
T2: SELECT * FROM orders WHERE status = 'NEW' FOR UPDATE;  - Locks different gaps
T1: INSERT INTO orders (status) VALUES ('NEW');  - Waits for T2's gap lock
-- Deadlock
```

PostgreSQL doesn't have gap locks (it uses MVCC), which is one reason I prefer it. If you're on MySQL, be aware of gap locking behavior and consider using READ COMMITTED isolation to reduce gap lock scope.

## Deadlock Detection and Debugging

### PostgreSQL

```sql
-- Check for current locks
SELECT pid, usename, query, wait_event_type, state
FROM pg_stat_activity
WHERE state = 'active';

-- Check lock waits
SELECT blocked_locks.pid AS blocked_pid,
       blocking_locks.pid AS blocking_pid,
       blocked_activity.query AS blocked_query,
       blocking_activity.query AS blocking_query
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_locks blocking_locks
    ON blocking_locks.locktype = blocked_locks.locktype
    AND blocking_locks.relation = blocked_locks.relation
    AND blocking_locks.pid != blocked_locks.pid
JOIN pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
JOIN pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;
```

PostgreSQL logs deadlocks automatically (with the queries involved) at the default log level. The log tells you exactly which queries were involved and which was killed. Always read the deadlock log - it's the fastest path to understanding the cause.

### MySQL

```sql
SHOW ENGINE INNODB STATUS;
```

The `LATEST DETECTED DEADLOCK` section shows the two transactions, the locks they held, and the locks they were waiting for. Parse it carefully - InnoDB's output is verbose but contains everything you need.

### Application-Level Logging

Add logging around lock acquisition so you can correlate application code with database locks:

```java
log.debug("Acquiring lock on order {}", orderId);
Order order = orderRepository.findByIdForUpdate(orderId);
log.debug("Acquired lock on order {}", orderId);
```

In a deadlock investigation, these logs tell you which code path was responsible, which is faster than tracing from SQL back to Java.

## Prevention Strategies

1. **Consistent lock ordering.** Always. Sort entities by ID before locking them.
2. **Keep transactions short.** Fewer locks held for less time means fewer deadlock opportunities.
3. **Lock the minimum.** Lock one row, not a range. Use specific WHERE clauses, not table scans.
4. **Use optimistic locking when contention is low.** No locks, no deadlocks.
5. **Set lock timeouts.** Don't let transactions wait forever. Fail fast and retry.
6. **Retry on deadlock.** The database picks a victim. Retry that transaction.

```java
@Retryable(
    retryFor = {CannotAcquireLockException.class, DeadlockLoserDataAccessException.class},
    maxAttempts = 3,
    backoff = @Backoff(delay = 100, multiplier = 2)
)
@Transactional
public void riskyOperation() { ... }
```

## The Hard Truth

You can't always prevent deadlocks. In a sufficiently complex system with enough concurrency, some deadlock patterns are unavoidable without restructuring the entire data access layer. What you can do is make deadlocks rare (consistent ordering, short transactions) and survivable (retries, timeouts, monitoring).

Monitor your deadlock rate. If it's zero, great. If it's one per day, probably fine - the retries handle it. If it's one per minute, you have a structural problem that needs architectural attention.

That 2 AM page? It was Pattern 1 - inconsistent lock ordering in a payment service. Two concurrent refunds for the same customer were locking accounts in different orders. The fix was three lines of code (sort by account ID before locking). The investigation was four hours. Write the fix first next time.
