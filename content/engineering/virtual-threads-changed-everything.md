+++
title = "Virtual Threads Changed How I Think About Concurrency"
date = 2026-04-03
[taxonomies]
tags = ["java", "concurrency", "spring-boot"]
+++

I've been a "thread pool of 200, tune it until it works" kind of developer for years. Virtual threads (Project Loom, available since Java 21) made me realize how much of that was just compensating for the fact that platform threads are expensive.

## The Old Way

In a typical Spring Boot service with a Tomcat thread pool, you get 200 threads by default. Each thread handles one request. If the request blocks on a database call or an HTTP call to another service, the thread sits there doing nothing. Your 200 threads aren't doing 200 things - they're waiting for 200 responses.

When all 200 threads are blocked waiting for I/O, your service stops accepting new requests. The standard fix: increase the thread pool. But platform threads cost ~1MB of stack memory each. 1000 threads = 1GB just for thread stacks. And the OS scheduler degrades with too many threads. There's a ceiling.

## What Virtual Threads Change

Virtual threads are cheap. You can have millions of them. They're managed by the JVM, not the OS. When a virtual thread blocks on I/O, the JVM unmounts it from the carrier thread and mounts another virtual thread. The carrier thread never actually blocks.

The practical effect: you don't need to think about thread pool sizing for I/O-bound workloads anymore. Each request gets its own virtual thread, and the JVM handles the multiplexing.

Enabling it in Spring Boot 3.2+:

```yaml
spring:
  threads:
    virtual:
      enabled: true
```

One line. Your Tomcat thread pool is replaced with virtual threads. Every request handler now runs on a virtual thread.

## Where It Actually Helps

The services where I've seen the biggest improvement are the ones that make multiple blocking calls per request - the typical "backend for frontend" pattern:

```java
@GetMapping("/order/{id}")
public OrderDetails getOrder(@PathVariable String id) {
    Order order = orderService.findById(id);           // DB call, 5ms
    Customer customer = customerApi.fetch(order.customerId());  // HTTP, 50ms
    List<Item> items = inventoryApi.fetchItems(order.itemIds()); // HTTP, 80ms
    ShippingStatus status = shippingApi.status(order.shipmentId()); // HTTP, 30ms

    return OrderDetails.from(order, customer, items, status);
}
```

With platform threads, this request holds a thread for ~165ms (sum of all calls). With 200 threads, you max out at ~1200 requests/second.

With virtual threads, the thread cost is negligible. The bottleneck moves to the actual I/O capacity and the downstream services, not your thread pool. In our testing, the same service handled 5x more concurrent requests before any other bottleneck appeared.

## Where It Doesn't Help

CPU-bound work. If your threads are doing computation, not waiting for I/O, virtual threads don't help. You're still limited by the number of CPU cores. For CPU-heavy work, the old approach of a fixed thread pool sized to the core count is still correct.

## The Gotchas

### Synchronized Blocks Pin the Carrier

If a virtual thread enters a `synchronized` block and then blocks on I/O inside it, the carrier thread gets pinned - it can't be reused for other virtual threads. This defeats the purpose.

The fix: replace `synchronized` with `ReentrantLock`:

```java
// Don't do this with virtual threads
synchronized (lock) {
    database.query(); // pins the carrier thread
}

// Do this instead
lock.lock();
try {
    database.query(); // virtual thread unmounts cleanly
} finally {
    lock.unlock();
}
```

Spring Boot and most modern libraries have fixed their `synchronized` blocks. But if you have custom code with `synchronized` + I/O, check it.

### Connection Pool Exhaustion

Virtual threads remove the thread pool bottleneck. Now the bottleneck is the connection pool. If you had 200 threads and a connection pool of 10, at most 10 threads were querying the database at once. The other 190 waited for connections, but that was fine because the thread pool was the real bottleneck.

With virtual threads, suddenly thousands of requests can reach the connection pool simultaneously. If your pool is sized at 10, you get connection timeout errors under load.

The fix isn't "make the pool bigger" (the database can only handle so many connections). The fix is to use a semaphore to limit concurrent database access:

```java
private final Semaphore dbSemaphore = new Semaphore(10);

public Result query() {
    dbSemaphore.acquire();
    try {
        return jdbcTemplate.query(...);
    } finally {
        dbSemaphore.release();
    }
}
```

Or just keep the connection pool sized appropriately and accept that `connection-timeout` will fire when the pool is full. The virtual threads waiting for a connection don't consume OS resources, so waiting is cheap.

## The Verdict

Virtual threads aren't magic. They don't make your code faster - they make your code scale better under I/O-bound workloads. For the kind of services I build (API layers that call databases and other services), they're a significant improvement with minimal code changes.

Enable them. Test under load. Watch for `synchronized` pinning and connection pool exhaustion. That's it.
