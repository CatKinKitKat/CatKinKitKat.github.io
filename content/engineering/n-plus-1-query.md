+++
title = "The N+1 Query Problem, or How Your ORM Is Lying to You"
date = 2025-06-10
[taxonomies]
tags = ["jpa", "hibernate", "performance"]
+++

The N+1 problem is the single most common performance issue in JPA applications. I've found it in every codebase I've ever audited. Every single one. It's so common that if someone tells me their Hibernate app is slow, I start looking for N+1 queries before I look at anything else.

## What It Is

You want to load 100 orders and their line items. Hibernate executes:

1. One query to load 100 orders
2. 100 queries to load line items - one per order

That's 101 queries when you needed 1 or 2. The "N" is the number of parent entities. The "+1" is the initial query to load them.

```java
@Entity
public class Order {
    @OneToMany(mappedBy = "order")
    private List<LineItem> items; // Default: LAZY fetch
}

// This triggers the N+1
List<Order> orders = orderRepository.findAll();
for (Order order : orders) {
    order.getItems().size(); // Each call fires a separate SELECT
}
```

Each `getItems()` call fires a SQL query because the items are lazily loaded. Hibernate fetches them one at a time, on demand. With 100 orders, you get 100 additional queries. With 10,000 orders, well, you get the picture.

## How to Detect It

### Enable SQL Logging

```yaml
spring:
  jpa:
    show-sql: true
    properties:
      hibernate:
        format_sql: true
```

Or better, use `hibernate.generate_statistics`:

```yaml
spring:
  jpa:
    properties:
      hibernate:
        generate_statistics: true
```

This gives you a summary at the end of each session: how many queries were executed, how many entities were loaded, how long it all took. If the query count is suspiciously high relative to the number of entities, you have N+1 problems.

### Use a Query Counter in Tests

This is the approach I prefer. Wrap your data source in a proxy that counts queries:

```java
@Test
void shouldLoadOrdersInTwoQueries() {
    QueryCounter counter = QueryCounter.start();

    List<Order> orders = orderService.findRecentOrders();
    orders.forEach(o -> o.getItems().size()); // Force lazy loading

    assertThat(counter.getCount()).isLessThanOrEqualTo(2);
}
```

Libraries like `datasource-proxy` or `p6spy` make this easy. I add query-count assertions to my integration tests for every repository method. It's the only reliable way to prevent N+1 regressions.

## Fix 1: JOIN FETCH

The most common fix. Tell Hibernate to load the association in the same query:

```java
@Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.status = :status")
List<Order> findByStatusWithItems(@Param("status") OrderStatus status);
```

This generates a single SQL query with a JOIN. One query, all the data. Problem solved.

Except: you can't JOIN FETCH two collections at once. If your Order has both `items` and `payments`, fetching both in one query produces a cartesian product and Hibernate will throw `MultipleBagFetchException` if both are `List` types. More on that later.

## Fix 2: Entity Graphs

Entity graphs let you declare which associations to fetch without hardcoding it in the query:

```java
@EntityGraph(attributePaths = {"items", "customer"})
List<Order> findByStatus(OrderStatus status);
```

This is equivalent to JOIN FETCH but defined declaratively. Spring Data JPA integrates with entity graphs natively. I prefer entity graphs when the same query is used with different fetch plans in different contexts.

You can also define named entity graphs on the entity:

```java
@NamedEntityGraph(
    name = "Order.withItemsAndCustomer",
    attributeNodes = {
        @NamedAttributeNode("items"),
        @NamedAttributeNode("customer")
    }
)
@Entity
public class Order { ... }
```

## Fix 3: Batch Fetching

If JOIN FETCH isn't practical (complex queries, pagination), batch fetching reduces N+1 to N/batch-size+1:

```java
@Entity
public class Order {
    @OneToMany(mappedBy = "order")
    @BatchSize(size = 25)
    private List<LineItem> items;
}
```

Now when Hibernate loads items for one order, it loads items for 25 orders at once using an `IN` clause:

```sql
SELECT * FROM line_items WHERE order_id IN (?, ?, ?, ... 25 ids)
```

100 orders now require 1 + 4 queries instead of 1 + 100. Not as good as JOIN FETCH, but works in situations where JOIN FETCH doesn't (like pagination).

You can also set a global default:

```yaml
spring:
  jpa:
    properties:
      hibernate:
        default_batch_fetch_size: 25
```

I set this globally on every Hibernate project. It's a safety net for cases where you forget to add a JOIN FETCH.

## Fix 4: DTO Projections

Sometimes you don't need the entities at all. A DTO projection avoids the N+1 problem entirely because there's no lazy loading to trigger:

```java
public record OrderSummary(String id, BigDecimal total, int itemCount) {}

@Query("""
    SELECT new com.example.OrderSummary(o.id, o.total, SIZE(o.items))
    FROM Order o
    WHERE o.status = :status
    """)
List<OrderSummary> findSummariesByStatus(@Param("status") OrderStatus status);
```

One query, no entities, no lazy loading, no N+1. For read-only use cases (dashboards, reports, APIs), this is the most efficient option.

## Verifying the Fix

After applying any fix, verify with two things:

1. **SQL logging** - count the queries. It should be 1-2, not N+1.
2. **Query count assertions in tests** - automated, runs on every build, catches regressions.

Don't trust "it feels faster." Measure. Count queries. Assert in tests. The N+1 problem has a way of creeping back in when someone adds a new association or changes a fetch plan.

## The Uncomfortable Truth

The N+1 problem is fundamentally a consequence of transparent lazy loading. The ORM makes it easy to traverse relationships without thinking about the SQL being generated. That's the feature. That's also the bug.

My advice: treat every association traversal as a potential N+1. Use `@BatchSize` globally as a safety net. Use JOIN FETCH and entity graphs for known hot paths. Assert query counts in tests. And occasionally, skip the ORM entirely and write a plain SQL query that does exactly what you need. There's no shame in that.
