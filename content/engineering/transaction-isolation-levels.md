+++
title = "Transaction Isolation Levels - The Practical Guide Nobody Gives You"
date = 2025-08-01
[taxonomies]
tags = ["database", "transactions", "java"]
+++

Transaction isolation levels are one of those topics where everyone has a vague understanding but nobody can explain the actual differences without a whiteboard. I've been that person. Let me try to fix that.

## The Problem

Multiple transactions running concurrently can interfere with each other. Without isolation, you get anomalies - situations where the data looks wrong depending on timing. The SQL standard defines four isolation levels, each preventing different anomalies.

## The Anomalies

### Dirty Read

Transaction A reads data that Transaction B has written but not committed. If B rolls back, A has read data that never existed.

```
T1: UPDATE accounts SET balance = 0 WHERE id = 1;
    - (not committed yet)
T2: SELECT balance FROM accounts WHERE id = 1;
    - Returns 0 (dirty read)
T1: ROLLBACK;
    - The balance was never actually 0
```

### Non-Repeatable Read

Transaction A reads the same row twice and gets different results because Transaction B committed a change between the reads.

```
T1: SELECT balance FROM accounts WHERE id = 1;  - Returns 100
T2: UPDATE accounts SET balance = 50 WHERE id = 1; COMMIT;
T1: SELECT balance FROM accounts WHERE id = 1;  - Returns 50 (changed!)
```

### Phantom Read

Transaction A runs the same query twice and gets different rows because Transaction B inserted or deleted rows between the queries.

```
T1: SELECT COUNT(*) FROM orders WHERE status = 'ACTIVE';  - Returns 10
T2: INSERT INTO orders (status) VALUES ('ACTIVE'); COMMIT;
T1: SELECT COUNT(*) FROM orders WHERE status = 'ACTIVE';  - Returns 11 (phantom)
```

### Write Skew

The sneaky one. Two transactions read the same data, make decisions based on it, and write changes that are individually valid but collectively wrong.

```
-- Rule: at least one doctor must be on call
-- Both doctors are on call

T1: SELECT COUNT(*) FROM doctors WHERE on_call = true;  - 2
T2: SELECT COUNT(*) FROM doctors WHERE on_call = true;  - 2
T1: UPDATE doctors SET on_call = false WHERE id = 1; COMMIT;  - Still 1 on call
T2: UPDATE doctors SET on_call = false WHERE id = 2; COMMIT;  - Now 0 on call!
```

Each transaction thought it was safe to remove one doctor because two were on call. Both committed. Nobody is on call. This is a write skew, and it's the most dangerous anomaly because it's invisible to each transaction individually.

## The Isolation Levels

### READ UNCOMMITTED

Allows dirty reads. Basically no isolation. I've never seen a legitimate use for this in application code. The only time I've seen it used intentionally was for monitoring queries that need to see in-flight data and don't care about consistency.

### READ COMMITTED

Prevents dirty reads. Each read sees only committed data. This is the default in PostgreSQL and Oracle.

The catch: you can still get non-repeatable reads and phantom reads. If your transaction reads the same row twice, it might get different values if another transaction committed between the reads.

```java
@Transactional(isolation = Isolation.READ_COMMITTED)
public void processOrder(String orderId) {
    // Each query sees the latest committed state
    // But two reads of the same row might differ
}
```

For most CRUD applications, READ COMMITTED is fine. Your code should already handle concurrent modifications (optimistic locking, retries), and non-repeatable reads are usually acceptable.

### REPEATABLE READ

Prevents dirty reads and non-repeatable reads. Once a transaction reads a row, re-reading it returns the same value, regardless of other transactions committing changes. This is the default in MySQL (InnoDB).

In PostgreSQL, REPEATABLE READ also prevents phantom reads (because PostgreSQL implements it using MVCC snapshots, which is more strict than the SQL standard requires).

In MySQL, REPEATABLE READ does NOT prevent phantom reads for locking reads (`SELECT ... FOR UPDATE`), only for plain SELECT. This distinction has bitten me.

```java
@Transactional(isolation = Isolation.REPEATABLE_READ)
public void generateReport(String period) {
    // All reads see a consistent snapshot
    // Good for reports that need consistency across multiple queries
}
```

### SERIALIZABLE

Prevents all anomalies, including write skew. Transactions behave as if they ran one at a time, in sequence. The database either uses actual serial execution or detects conflicts and aborts one of the conflicting transactions.

```java
@Transactional(isolation = Isolation.SERIALIZABLE)
public void transferFunds(String from, String to, BigDecimal amount) {
    // Fully isolated - no concurrency anomalies possible
    // But may throw serialization failures that require retry
}
```

The cost: reduced concurrency and serialization failures. When two serializable transactions conflict, one gets aborted with a serialization error. Your code must be ready to retry.

## MVCC: How It Actually Works

Most modern databases (PostgreSQL, MySQL InnoDB, Oracle) implement isolation using Multi-Version Concurrency Control (MVCC). Instead of locking rows, the database keeps multiple versions of each row. Each transaction sees a snapshot of the data as of its start time (or statement start time, depending on the level).

**READ COMMITTED**: snapshot is taken at the start of each statement. Each query sees the latest committed data at the moment it runs.

**REPEATABLE READ**: snapshot is taken at the start of the transaction. All queries in the transaction see the same data, regardless of commits by other transactions.

**SERIALIZABLE** (PostgreSQL): same as REPEATABLE READ, but the database also tracks read dependencies and aborts transactions that would produce anomalies.

The beauty of MVCC: readers don't block writers, and writers don't block readers. A long-running report doesn't lock the tables it's reading. A batch update doesn't block queries. The trade-off is that the database must keep old row versions around until no transaction needs them (vacuum in PostgreSQL).

## Which Level to Actually Use

**Default to READ COMMITTED.** It's the PostgreSQL default for a reason. It prevents dirty reads, works well with optimistic locking (`@Version`), and doesn't surprise you with serialization failures.

**Use REPEATABLE READ for:**
- Reports that need consistency across multiple queries
- Batch jobs that read and process data over multiple steps
- Anything where "the data changed between my two queries" is a bug

**Use SERIALIZABLE for:**
- Financial transactions where write skew is unacceptable
- Inventory systems where overselling is worse than reduced throughput
- Any business logic where correctness trumps performance

But - and this is important - if you use SERIALIZABLE, you MUST implement retry logic:

```java
@Retryable(
    retryFor = {CannotAcquireLockException.class, OptimisticLockingFailureException.class},
    maxAttempts = 3,
    backoff = @Backoff(delay = 100, multiplier = 2)
)
@Transactional(isolation = Isolation.SERIALIZABLE)
public void criticalBusinessLogic() {
    // ...
}
```

Without retries, serialization failures become user-visible errors. With retries, they're invisible performance overhead.

## The Spring Boot Configuration

Set the default isolation level for the entire application:

```yaml
spring:
  datasource:
    hikari:
      transaction-isolation: TRANSACTION_READ_COMMITTED
```

Or override per-method with `@Transactional(isolation = ...)`. I set the global default to READ COMMITTED (which is already the PostgreSQL default) and only escalate to higher levels for specific methods that need it.

## The Uncomfortable Truth

Most developers don't think about isolation levels until something goes wrong. And most of the time, READ COMMITTED with optimistic locking handles the concurrency correctly. But when it doesn't - when you get a write skew in a payment system or a phantom read in an inventory check - the consequences are painful and the debugging is harder because the issue is intermittent and timing-dependent.

Understand the anomalies. Know your defaults. And when in doubt, write a test that runs concurrent transactions and checks for the anomaly you're worried about. It's the only way to be sure.
