+++
title = "The Hibernate Performance Tuning Checklist I Actually Use"
date = 2025-07-15
[taxonomies]
tags = ["hibernate", "performance", "database"]
+++

Hibernate performance tuning is 80% "stop doing dumb things" and 20% "enable the features that are off by default for some reason." Here's the checklist I run through on every project, in order of impact.

## 1. Enable Statistics First

You can't optimize what you can't measure. Turn on Hibernate statistics:

```yaml
spring:
  jpa:
    properties:
      hibernate:
        generate_statistics: true
```

This logs a summary per session: queries executed, entities loaded, cache hits/misses, and time spent. If you see 200 queries for a single request, you have a problem. If you see 2, you probably don't.

For production, don't keep text logging enabled. Export the statistics to Micrometer/Prometheus instead:

```java
@Bean
public HibernateMetricsExporter hibernateMetrics(EntityManagerFactory emf) {
    return new HibernateMetricsExporter(emf, meterRegistry);
}
```

## 2. Fix N+1 Queries

Covered in detail in my N+1 post, but for the checklist:

- Set global batch fetch size: `hibernate.default_batch_fetch_size = 25`
- Use JOIN FETCH for known hot paths
- Use entity graphs for flexible fetch plans
- Add query count assertions to integration tests
- Set all `@ManyToOne` and `@OneToOne` to `FetchType.LAZY`

This single item accounts for 80% of all Hibernate performance issues I've seen.

## 3. JDBC Batching

By default, Hibernate sends INSERT and UPDATE statements one at a time. If you're saving 100 entities, that's 100 round trips to the database.

```yaml
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 25
        order_inserts: true
        order_updates: true
```

`batch_size` tells Hibernate to group statements into batches. `order_inserts` and `order_updates` sort the statements by entity type, which is required for batching to work across entity types.

The difference is dramatic. Saving 1000 entities goes from 1000 round trips to 40. I've seen batch insert performance improve 10x just by adding these three properties.

One gotcha: batching doesn't work if your entities use `IDENTITY` id generation strategy. Hibernate needs to disable batching to get the generated IDs back immediately. Use `SEQUENCE` with an allocation size instead:

```java
@Id
@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "order_seq")
@SequenceGenerator(name = "order_seq", sequenceName = "order_seq", allocationSize = 50)
private Long id;
```

The `allocationSize` reduces the number of sequence calls. Hibernate pre-allocates 50 IDs at a time, so you don't hit the database for every single insert.

## 4. Statement Caching

PreparedStatement caching avoids re-parsing SQL on every execution. Configure it at the JDBC driver level:

For HikariCP with PostgreSQL:

```yaml
spring:
  datasource:
    hikari:
      data-source-properties:
        prepStmtCacheSize: 250
        prepStmtCacheSqlLimit: 2048
        cachePrepStmts: true
```

For PostgreSQL specifically, also enable server-side prepared statements:

```yaml
spring:
  datasource:
    hikari:
      data-source-properties:
        prepareThreshold: 5
```

This tells the JDBC driver to use server-side prepared statements after a query has been executed 5 times. The database parses and plans the query once, then reuses the plan. For frequently executed queries, this eliminates parsing overhead.

## 5. Second-Level Cache

Hibernate's first-level cache (the session cache) lives for the duration of a transaction. The second-level cache (L2) persists across transactions and sessions.

```yaml
spring:
  jpa:
    properties:
      hibernate:
        cache:
          use_second_level_cache: true
          region.factory_class: org.hibernate.cache.jcache.JCacheRegionFactory
      javax:
        cache:
          provider: org.ehcache.jsr107.EhcacheCachingProvider
```

Annotate entities you want cached:

```java
@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Country {
    // Rarely changes, frequently read - good candidate
}
```

Good candidates for L2 cache:
- Reference data (countries, currencies, status codes)
- Configuration entities that rarely change
- Entities read frequently by ID

Bad candidates:
- Entities that change frequently
- Entities with large collections
- Anything where stale data is unacceptable

I use L2 cache selectively. Caching everything is worse than caching nothing because the cache invalidation overhead can exceed the query cost.

## 6. Query Cache

The query cache stores the results of queries (as entity IDs), not just individual entities. It works in combination with the L2 entity cache.

```yaml
spring:
  jpa:
    properties:
      hibernate:
        cache:
          use_query_cache: true
```

Then mark individual queries as cacheable:

```java
@QueryHints(@QueryHint(name = "org.hibernate.cacheable", value = "true"))
List<Country> findAll();
```

The query cache is invalidated when any entity in the cached query's table is modified. This makes it effective only for tables that are read-heavy and rarely written. For most transactional tables, the query cache does more harm than good.

## 7. Slow Query Log

Enable Hibernate's slow query log to catch expensive queries before they become incidents:

```yaml
spring:
  jpa:
    properties:
      hibernate:
        session:
          events:
            log:
              LOG_QUERIES_SLOWER_THAN_MS: 200
```

Any query taking longer than 200ms gets logged. Adjust the threshold based on your SLAs. I usually start at 500ms and tighten it as the application matures.

## 8. Batch Operations for Bulk Work

For bulk updates or deletes, don't load entities into memory just to modify them:

```java
// Don't do this - loads every entity into memory
List<Order> orders = orderRepository.findByStatus(EXPIRED);
orders.forEach(o -> o.setStatus(ARCHIVED));
// Hibernate generates N UPDATE statements

// Do this instead
@Modifying
@Query("UPDATE Order o SET o.status = 'ARCHIVED' WHERE o.status = 'EXPIRED'")
int archiveExpiredOrders();
```

The modifying query updates directly in the database. No entity loading, no dirty checking, no individual UPDATE statements. For bulk operations, this is orders of magnitude faster.

For more complex bulk operations, use `StatelessSession`:

```java
StatelessSession session = sessionFactory.openStatelessSession();
Transaction tx = session.beginTransaction();

ScrollableResults results = session.createQuery("FROM Order WHERE status = 'EXPIRED'")
    .scroll(ScrollMode.FORWARD_ONLY);

while (results.next()) {
    Order order = (Order) results.get(0);
    order.setStatus(OrderStatus.ARCHIVED);
    session.update(order);
}

tx.commit();
session.close();
```

`StatelessSession` bypasses the first-level cache, dirty checking, and event listeners. It's a thin wrapper around JDBC. Use it for batch processing, data migrations, and anything where the full ORM overhead isn't needed.

## 9. Read-Only Transactions

For queries that don't modify data, mark the transaction as read-only:

```java
@Transactional(readOnly = true)
public List<OrderSummary> getRecentOrders() {
    return orderRepository.findRecent();
}
```

This tells Hibernate to skip dirty checking for entities loaded in this transaction. No snapshot comparison on flush, no memory overhead for tracking changes. For read-heavy services, this is a meaningful optimization.

## 10. Connection Pool Monitoring

This isn't strictly Hibernate, but HikariCP metrics catch connection-related performance issues:

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 10
      minimum-idle: 5
      leak-detection-threshold: 30000
```

`leak-detection-threshold` logs a warning if a connection is checked out for more than 30 seconds. This catches the "someone forgot to close a session" or "query running too long" problems before they exhaust the pool.

## The Order Matters

Run through this checklist in order. Items 1-3 (statistics, N+1 fixes, JDBC batching) solve 90% of Hibernate performance issues. Items 4-6 (statement caching, L2 cache, query cache) are optimizations for specific workloads. Items 7-10 are monitoring and best practices.

Don't start with the second-level cache when your queries are doing N+1. Don't tune connection pool sizes when you're sending 500 unneeded queries per request. Fix the fundamentals first.
