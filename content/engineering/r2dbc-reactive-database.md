+++
title = "R2DBC: Reactive Database Access and Its Discontents"
date = 2026-03-09
[taxonomies]
tags = ["java", "reactive", "database"]
+++

Here's the thing about reactive programming: the entire chain is only as non-blocking as its weakest link. You can have the most beautifully composed Reactor pipeline in the world, and if it bottlenecks on a JDBC call wrapped in `Schedulers.boundedElastic()`, you've just built a Rube Goldberg machine that does what a Servlet would have done with less code.

R2DBC exists to fix that. Whether it succeeds is... complicated.

## What R2DBC Is

R2DBC (Reactive Relational Database Connectivity) is a specification for non-blocking database drivers. Where JDBC blocks on every query, R2DBC returns `Publisher` types (Mono/Flux in Reactor land) and never blocks the calling thread.

```java
// JDBC - blocks the thread
User user = jdbcTemplate.queryForObject(
    "SELECT * FROM users WHERE id = ?", userRowMapper, id);

// R2DBC - non-blocking
Mono<User> user = databaseClient.sql("SELECT * FROM users WHERE id = :id")
    .bind("id", id)
    .map(row -> new User(row.get("id", Long.class), row.get("name", String.class)))
    .one();
```

The thread that executes the R2DBC query doesn't wait for the database response. It goes back to the event loop and picks up other work. When the response arrives, a thread processes the result and continues the chain.

## Spring Data R2DBC

If you've used Spring Data JPA, Spring Data R2DBC looks familiar, until it doesn't:

```java
public interface UserRepository extends ReactiveCrudRepository<User, Long> {
    Flux<User> findByLastName(String lastName);
    Mono<User> findByEmail(String email);
}
```

The repository interface extends `ReactiveCrudRepository` instead of `JpaRepository`, and return types are `Mono`/`Flux`. Derived query methods work the same way.

Here's where the familiarity ends. Spring Data R2DBC is *not* Spring Data JPA with reactive return types. There is no:

- Lazy loading. Forget `@OneToMany(fetch = FetchType.LAZY)`. R2DBC doesn't do entity relationships at all by default.
- Entity graph. No `@EntityGraph`, no join fetching.
- First-level cache. No persistence context, no dirty checking.
- Schema generation. No `ddl-auto=update`.
- Complex joins through method names. `findByOrderCustomerName` won't generate a join.

What you get is essentially a reactive query runner with entity mapping. Which, honestly, is fine for a lot of use cases. But if you're coming from JPA expecting the same level of ORM magic, you're going to have a bad time.

For relationships, you compose queries manually:

```java
public Mono<OrderDetails> getOrderWithItems(Long orderId) {
    return orderRepository.findById(orderId)
        .flatMap(order -> itemRepository.findByOrderId(orderId)
            .collectList()
            .map(items -> new OrderDetails(order, items)));
}
```

It's explicit. Some people call that a feature. I call it "I miss JPA but I understand why we can't have it here."

## The Connection Pool

R2DBC has its own connection pool implementation (`r2dbc-pool`) separate from HikariCP. Configuration lives in your Spring properties:

```yaml
spring:
  r2dbc:
    url: r2dbc:postgresql://localhost:5432/mydb
    username: app
    password: secret
    pool:
      initial-size: 5
      max-size: 20
      max-idle-time: 30m
```

One thing I've learned the hard way: R2DBC pool sizes should generally be smaller than what you'd use with JDBC. With JDBC, you need a connection per blocked thread. With R2DBC, connections are multiplexed more efficiently because nothing blocks. A pool of 20 R2DBC connections can serve the same throughput as 200 JDBC connections in some workloads.

## Reactive Elasticsearch

If you're in the Spring ecosystem, Spring Data also supports reactive Elasticsearch through `ReactiveElasticsearchClient`:

```java
public interface ProductSearchRepository
    extends ReactiveElasticsearchRepository<Product, String> {
    Flux<Product> findByNameContaining(String query);
}
```

Under the hood, it uses the non-blocking HTTP client to talk to Elasticsearch. This is one of the places where reactive actually makes a lot of sense - search queries can be slow and bursty, and tying up a thread per search query on a busy service is wasteful.

The reactive Elasticsearch client has improved significantly in recent versions. Early versions were rough - missing features, inconsistent error handling. It's usable now. Not JPA-level polished, but usable.

## When R2DBC Makes Sense

Here's my honest assessment:

**Good fit:**
- API gateway or aggregation layer with high concurrent connections
- Services where database access is one part of a fully reactive pipeline
- Streaming query results (Flux) for large result sets without buffering everything in memory
- Services already committed to WebFlux

**Not a good fit:**
- Traditional CRUD apps (just use JDBC + virtual threads)
- Teams without reactive experience (the learning curve is steep and the failure modes are subtle)
- Complex domain models with many relationships (you'll miss JPA)
- Services that also need to call JDBC-based libraries (mixing reactive and blocking defeats the purpose)

## The Maturity Gap

Let's be blunt: R2DBC is less mature than JDBC. JDBC has been around since 1997. R2DBC hit 1.0 in 2022. The gap shows:

**Driver support:** PostgreSQL and MySQL/MariaDB drivers are solid. Oracle has an R2DBC driver. SQL Server's driver exists but I've hit quirks. H2 works for testing.

**Tooling:** Most database migration tools (Flyway, Liquibase) still use JDBC under the hood. You end up with both JDBC and R2DBC drivers on the classpath for migrations alone. It works, but it feels wrong.

**Debugging:** When something goes wrong with an R2DBC query, the stack trace is a reactive mess. Add Reactor's debugging tools (`ReactorDebugAgent`) or you'll be reading Netty internals trying to figure out which query failed.

**Connection pool monitoring:** HikariCP has excellent metrics integration. The R2DBC pool has metrics too, but the ecosystem of dashboards and alerts built around HikariCP doesn't exist for R2DBC yet.

**ORM features:** As mentioned, no lazy loading, no dirty checking, no schema generation. If you need these, you're stuck with JPA or doing it by hand.

## Reactive Logging Patterns

Logging in reactive code is its own adventure. MDC (Mapped Diagnostic Context), that thing you use for request correlation IDs, is ThreadLocal-based. In a reactive pipeline, your code hops between threads, so MDC values vanish.

Reactor has `Context` for this, and you can bridge it to MDC with some plumbing:

```java
// Set context at the edge (e.g., WebFilter)
@Component
public class CorrelationIdFilter implements WebFilter {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
        String correlationId = exchange.getRequest().getHeaders()
            .getFirst("X-Correlation-ID");
        return chain.filter(exchange)
            .contextWrite(Context.of("correlationId",
                correlationId != null ? correlationId : UUID.randomUUID().toString()));
    }
}
```

Then use `doOnEach` or a hook to copy Reactor Context to MDC before each signal:

```java
Hooks.onEachOperator(Operators.lift((scannable, subscriber) ->
    new CoreSubscriber<Object>() {
        @Override
        public void onNext(Object t) {
            copyContextToMdc(subscriber.currentContext());
            subscriber.onNext(t);
        }
        // ... implement other methods similarly
    }
));
```

It's boilerplate. Micrometer's context propagation library helps, and Spring Boot 3.x has improved this with automatic context propagation. But it's still more work than just using ThreadLocal MDC in a Servlet stack.

## The Virtual Threads Elephant in the Room

I keep coming back to this, and I'll be direct: for most Spring Boot services, Java 21 virtual threads make R2DBC unnecessary. Virtual threads make blocking calls cheap. A JDBC call on a virtual thread doesn't waste OS resources. You get the throughput benefits of non-blocking without rewriting your data access layer.

R2DBC still wins for streaming large result sets (Flux gives you genuine backpressure against the database) and for applications that are fully reactive end-to-end. But the "we need R2DBC for throughput" argument lost most of its weight when virtual threads landed.

If you're starting a new project in 2026 and wondering whether to use R2DBC or JDBC + virtual threads: use JDBC. Your team will be more productive, your debugging will be easier, and the JPA ecosystem is vastly more mature. Save R2DBC for the cases where you genuinely need backpressure or are already deep in the reactive ecosystem.

That's not a popular opinion in some circles, but I've done it both ways. The boring choice is usually the right one.
