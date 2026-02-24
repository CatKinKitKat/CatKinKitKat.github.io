+++
title = "Spring @Transactional: Mistakes Everyone Makes"
date = 2026-02-24
[taxonomies]
tags = ["spring-boot", "database", "transactions"]
+++

`@Transactional` is the most deceptively simple annotation in Spring. You put it on a method, and Spring wraps it in a database transaction. Done.

Except it's not done. The number of subtle bugs I've found involving `@Transactional` is genuinely impressive. Here are the mistakes I keep seeing - and making, if I'm being honest.

## The Self-Invocation Trap

This is the number one `@Transactional` bug, and it's bitten every Spring developer at least once.

```java
@Service
public class OrderService {

    public void processOrder(Order order) {
        // Some logic...
        saveOrder(order); // THIS DOES NOT RUN IN A TRANSACTION
    }

    @Transactional
    public void saveOrder(Order order) {
        orderRepository.save(order);
        auditRepository.save(new AuditEntry(order));
    }
}
```

When `processOrder` calls `saveOrder`, it's a direct method call within the same class. Spring's `@Transactional` works through AOP proxies - the proxy intercepts calls *from outside the class*. Internal calls bypass the proxy entirely, so the `@Transactional` annotation is silently ignored.

Your code looks transactional. It isn't. The order saves but the audit entry fails? Too bad, they're not in the same transaction.

Fixes:

```java
// Fix 1: Move the transactional method to a separate class
@Service
public class OrderPersistenceService {
    @Transactional
    public void saveOrder(Order order) {
        orderRepository.save(order);
        auditRepository.save(new AuditEntry(order));
    }
}

// Fix 2: Inject self (ugly but works)
@Service
public class OrderService {
    @Lazy @Autowired
    private OrderService self;

    public void processOrder(Order order) {
        self.saveOrder(order); // Goes through proxy
    }

    @Transactional
    public void saveOrder(Order order) {
        orderRepository.save(order);
        auditRepository.save(new AuditEntry(order));
    }
}
```

I prefer Fix 1. It's cleaner and more obvious. Fix 2 works but the self-injection pattern raises eyebrows in code reviews for good reason.

## Transaction Propagation

The `propagation` attribute controls what happens when a transactional method calls another transactional method. The default is `REQUIRED`, which joins the existing transaction if one exists.

```java
@Transactional // Default: REQUIRED
public void createOrder(Order order) {
    orderRepository.save(order);
    notificationService.sendOrderConfirmation(order); // What happens here?
}

@Transactional // Also REQUIRED
public void sendOrderConfirmation(Order order) {
    // This runs in the SAME transaction as createOrder
    notificationRepository.save(new Notification(order));
    emailClient.send(order.getEmail()); // If this fails...
}
```

With `REQUIRED`, both methods share the same transaction. If `emailClient.send()` throws an exception, the entire transaction rolls back - including the order creation. Is that what you want? Probably not. A failed email shouldn't prevent order creation.

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void sendOrderConfirmation(Order order) {
    // This runs in its OWN transaction
    notificationRepository.save(new Notification(order));
    emailClient.send(order.getEmail());
}
```

`REQUIRES_NEW` suspends the outer transaction and creates a new one. If the notification fails, only the notification transaction rolls back. The order creation transaction continues independently.

Other propagation values I use:

- `MANDATORY` - Requires an existing transaction. Throws if none exists. Good for methods that should never be called outside a transaction.
- `NOT_SUPPORTED` - Suspends any existing transaction. Useful for read-only operations that don't need transactional guarantees and might run long.

## The Read-Only Optimization

```java
@Transactional(readOnly = true)
public List<Order> getOrdersForCustomer(String customerId) {
    return orderRepository.findByCustomerId(customerId);
}
```

`readOnly = true` is not just documentation. It tells Hibernate to skip dirty checking on all loaded entities. In a query that returns 1000 entities, that's 1000 fewer objects that Hibernate needs to compare at flush time. It also enables the database to optimize the connection for read-only operations.

Always use `readOnly = true` for queries. It's free performance.

But - and this catches people - `readOnly = true` doesn't prevent writes at the database level. If your code calls `repository.save()` inside a `readOnly` transaction, the behavior depends on the JPA provider and database. Some will silently succeed. Some will throw. Don't rely on it as a safety mechanism.

## Connection Management

Every `@Transactional` method acquires a database connection from the pool for the duration of the transaction. Long-running transactions hold connections for a long time. If your pool has 10 connections and you have 10 threads in long transactions, every other request is blocked.

```java
// BAD: Holds a database connection while waiting for an HTTP response
@Transactional
public Order createOrder(CreateOrderRequest request) {
    Order order = orderRepository.save(new Order(request));
    PaymentResult payment = paymentClient.charge(request.getPaymentInfo()); // HTTP call!
    order.setPaymentId(payment.getId());
    orderRepository.save(order);
    return order;
}
```

That HTTP call to the payment service might take 2 seconds. That's 2 seconds of a database connection being held for nothing. Under load, this exhausts the connection pool.

```java
// BETTER: Minimize transaction scope
public Order createOrder(CreateOrderRequest request) {
    Order order = saveInitialOrder(request);
    PaymentResult payment = paymentClient.charge(request.getPaymentInfo());
    return updateOrderWithPayment(order.getId(), payment.getId());
}

@Transactional
private Order saveInitialOrder(CreateOrderRequest request) {
    return orderRepository.save(new Order(request));
}

@Transactional
private Order updateOrderWithPayment(String orderId, String paymentId) {
    Order order = orderRepository.findById(orderId).orElseThrow();
    order.setPaymentId(paymentId);
    return orderRepository.save(order);
}
```

Wait - those private methods won't work because of the self-invocation trap. You'd need to extract them to a separate service or use the self-injection pattern. This is one reason I sometimes prefer programmatic transaction management for complex flows:

```java
public Order createOrder(CreateOrderRequest request) {
    Order order = transactionTemplate.execute(status -> {
        return orderRepository.save(new Order(request));
    });

    PaymentResult payment = paymentClient.charge(request.getPaymentInfo());

    return transactionTemplate.execute(status -> {
        Order existing = orderRepository.findById(order.getId()).orElseThrow();
        existing.setPaymentId(payment.getId());
        return orderRepository.save(existing);
    });
}
```

`TransactionTemplate` gives you explicit control over transaction boundaries without the proxy gotchas.

## MDC Logging and Transactions

When debugging transaction issues, MDC (Mapped Diagnostic Context) logging is invaluable. Add the transaction ID to your log context so you can trace which operations happen in which transaction.

```java
@Aspect
@Component
public class TransactionLoggingAspect {

    @Around("@annotation(org.springframework.transaction.annotation.Transactional)")
    public Object logTransaction(ProceedingJoinPoint joinPoint) throws Throwable {
        String txId = UUID.randomUUID().toString().substring(0, 8);
        MDC.put("txId", txId);

        String method = joinPoint.getSignature().toShortString();
        log.debug("Transaction START: {} [tx:{}]", method, txId);

        try {
            Object result = joinPoint.proceed();
            log.debug("Transaction COMMIT: {} [tx:{}]", method, txId);
            return result;
        } catch (Exception e) {
            log.warn("Transaction ROLLBACK: {} [tx:{}] cause: {}",
                method, txId, e.getMessage());
            throw e;
        } finally {
            MDC.remove("txId");
        }
    }
}
```

With this in your logback pattern (`%X{txId}`), every log line within a transaction shows its transaction ID. When debugging "why did this rollback?", you can filter logs by transaction ID and see every operation that happened.

## Exception Handling

By default, `@Transactional` rolls back on unchecked exceptions (RuntimeException and its subclasses) and does NOT roll back on checked exceptions.

```java
@Transactional
public void processOrder(Order order) throws InsufficientStockException {
    orderRepository.save(order);
    inventoryService.reserve(order); // Throws InsufficientStockException (checked)
    // Transaction COMMITS despite the exception!
}
```

This surprises everyone. A checked exception propagates up, but the transaction commits. Your order is saved even though inventory reservation failed.

```java
// Fix: Explicitly roll back on checked exceptions
@Transactional(rollbackFor = InsufficientStockException.class)
public void processOrder(Order order) throws InsufficientStockException {
    orderRepository.save(order);
    inventoryService.reserve(order);
}

// Or: roll back on all exceptions
@Transactional(rollbackFor = Exception.class)
```

My rule: always specify `rollbackFor = Exception.class` unless you have a specific reason not to. The default behavior of committing on checked exceptions is a footgun that has caused real data integrity bugs in every codebase I've worked on.

## Testing Transactions

Spring's `@Transactional` in tests rolls back after each test by default. This keeps your test database clean but masks a subtle issue: it means your tests never actually commit, so any constraints checked at commit time (deferred constraints, certain triggers) are never validated.

```java
@SpringBootTest
@Transactional // Rolls back after each test - convenient but risky
class OrderServiceTest {

    @Test
    void createOrder_savesSuccessfully() {
        orderService.createOrder(request);
        // Transaction hasn't committed yet, deferred constraints not checked
    }
}
```

For integration tests that verify transactional behavior, I use `@Commit` or avoid `@Transactional` on the test class entirely and clean up manually.

## The Summary

1. **Self-invocation bypasses @Transactional.** Extract transactional logic to a separate bean.
2. **Propagation matters.** Use `REQUIRES_NEW` when a failure in one operation shouldn't roll back another.
3. **Use readOnly = true** for all read operations. Free performance.
4. **Minimize transaction scope.** Don't hold connections during I/O.
5. **Specify rollbackFor = Exception.class** to avoid silent commits on checked exceptions.
6. **Use TransactionTemplate** when you need explicit control.

`@Transactional` looks simple. The bugs it produces when misused are anything but.
