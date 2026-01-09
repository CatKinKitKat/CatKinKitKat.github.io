+++
title = "EAGER vs LAZY Fetching: A Story of Regret"
date = 2025-07-01
[taxonomies]
tags = ["jpa", "hibernate", "performance"]
+++

If I had a euro for every time EAGER fetching caused a production incident, I could retire. EAGER is the fetching strategy that looks convenient in development and becomes a catastrophe at scale. Let me tell you why LAZY should be your default for everything, always, no exceptions.

## The Basics

JPA has two fetch strategies:

- **EAGER** - load the association immediately when the parent entity is loaded
- **LAZY** - load the association only when it's accessed

```java
@ManyToOne(fetch = FetchType.EAGER)  // Loaded with every query
private Customer customer;

@OneToMany(fetch = FetchType.LAZY)   // Loaded only when accessed
private List<LineItem> items;
```

The JPA defaults are:
- `@ManyToOne` and `@OneToOne`: **EAGER** (this is the problem)
- `@OneToMany` and `@ManyToMany`: LAZY

Yes, `@ManyToOne` defaults to EAGER. This means every time you load an Order, the Customer is loaded too. Every time you load a LineItem, the Order and its Customer are loaded. You can see how this cascades into a tree of JOINs that fetches half your database for a simple query.

## Why EAGER Is a Code Smell

### You Can't Un-EAGER

Once a relationship is marked EAGER, every query that touches that entity loads the association. There's no way to say "load this entity but skip the eager association for this particular use case." You've made a global decision that applies everywhere.

With LAZY, you can choose to fetch when needed (JOIN FETCH, entity graph). With EAGER, you fetch always. The asymmetry is the problem: LAZY gives you a choice, EAGER takes it away.

### The Cascade of JOINs

```java
@Entity
public class Order {
    @ManyToOne(fetch = FetchType.EAGER)
    private Customer customer;
}

@Entity
public class Customer {
    @ManyToOne(fetch = FetchType.EAGER)
    private Company company;
}

@Entity
public class Company {
    @ManyToOne(fetch = FetchType.EAGER)
    private Address headquarters;
}
```

Load an Order, and Hibernate loads the Customer, the Company, and the Address. Every time. Even if you just want the order total. The SQL becomes a multi-table JOIN that grows with your entity graph.

I once profiled a service where loading a single entity triggered 12 JOINs because of cascading EAGER relationships. The query took 200ms for a single row. Changing everything to LAZY and adding targeted JOIN FETCHes dropped it to 3ms.

### N+1 With EAGER

Here's what surprises people: EAGER doesn't prevent N+1 queries. If you use `findAll()` from Spring Data, Hibernate loads the parent entities with one query, then fetches each EAGER association with additional queries. It's N+1, but now you can't fix it by making it LAZY because something somewhere depends on the eager loading.

JOIN FETCH only works in JPQL/HQL queries. For repository methods derived from method names (`findByStatus`), EAGER associations are fetched with separate queries.

## The LazyInitializationException

The reason people reach for EAGER in the first place:

```
org.hibernate.LazyInitializationException: could not initialize proxy - no Session
```

This happens when you access a LAZY association outside of a Hibernate session (transaction). The entity was loaded inside a transaction, the transaction closed, and now you're trying to traverse a lazy relationship. Hibernate can't load it because there's no database connection.

The wrong fix: make it EAGER. The right fixes:

### 1. Fetch What You Need in the Service Layer

```java
@Transactional(readOnly = true)
public OrderDetails getOrderDetails(String id) {
    Order order = orderRepository.findByIdWithItems(id); // JOIN FETCH
    return OrderDetails.from(order); // Access items inside the transaction
}
```

### 2. Use Entity Graphs

```java
@EntityGraph(attributePaths = {"items", "customer"})
Optional<Order> findById(String id);
```

### 3. Use DTO Projections

```java
@Query("SELECT new com.example.OrderSummary(o.id, o.total, c.name) " +
       "FROM Order o JOIN o.customer c WHERE o.id = :id")
OrderSummary findSummaryById(@Param("id") String id);
```

### 4. Do NOT Use Open Session in View

Spring Boot has `spring.jpa.open-in-view=true` by default. This keeps the Hibernate session open for the entire HTTP request, allowing lazy loading in the view/controller layer. It "solves" LazyInitializationException by keeping the session open.

It also means your database connection is held for the entire request duration, including response serialization. Under load, you run out of connections. Turn it off:

```yaml
spring:
  jpa:
    open-in-view: false
```

Yes, you'll get more LazyInitializationExceptions initially. Fix them properly (JOIN FETCH, entity graphs, DTOs). Don't paper over them with an open session.

## The MultipleBagFetchException

You can't JOIN FETCH two `List` collections in the same query:

```java
@Query("SELECT o FROM Order o JOIN FETCH o.items JOIN FETCH o.payments")
List<Order> findAllWithDetails();
// MultipleBagFetchException: cannot simultaneously fetch multiple bags
```

Hibernate calls `List` associations "bags" (unordered collections that allow duplicates). Fetching two bags produces a cartesian product that Hibernate can't deduplicate.

Fixes:

### 1. Use Set Instead of List

```java
@OneToMany(mappedBy = "order")
private Set<LineItem> items;  // Set, not List - no MultipleBagFetchException
```

This is my preferred fix when ordering doesn't matter.

### 2. Fetch in Multiple Queries

```java
@Transactional(readOnly = true)
public Order getOrderWithDetails(String id) {
    Order order = orderRepository.findByIdWithItems(id);     // JOIN FETCH items
    Hibernate.initialize(order.getPayments());                // Second query for payments
    return order;
}
```

Two queries instead of one, but no cartesian product.

### 3. Use @BatchSize

```java
@OneToMany(mappedBy = "order")
@BatchSize(size = 25)
private List<Payment> payments;
```

Batch fetching loads the association for multiple parent entities at once. It's not as efficient as JOIN FETCH, but it avoids the MultipleBagFetchException.

## My Rules

1. **Everything is LAZY.** No exceptions. Override `@ManyToOne` and `@OneToOne` defaults.
2. **Open Session in View is OFF.**
3. **Every query that needs associations uses JOIN FETCH or entity graphs.**
4. **Global `@BatchSize` of 25** as a safety net for forgotten fetch plans.
5. **Query count assertions** in integration tests for all repository methods.

These five rules have eliminated every Hibernate performance issue related to fetching in every project I've applied them to. They're not complex. They're not clever. But they work.
