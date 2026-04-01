+++
title = "Strangler Fig in Practice: Wrapping Legacy Services"
date = 2026-04-01
[taxonomies]
tags = ["migration", "legacy", "microservices", "patterns"]
+++

I mentioned the strangler fig pattern in my [WebLogic migration post](@/engineering/killing-weblogic.md) without going deep. Since it's the single most useful pattern for legacy migrations, it deserves its own post.

## The Idea

A strangler fig is a tree that grows around another tree, eventually replacing it entirely. The old tree doesn't get cut down - it gets gradually absorbed. Same principle for legacy systems.

Instead of rewriting the legacy system from scratch (which fails spectacularly more often than anyone admits), you build new functionality alongside it. New requests go to the new system; old requests keep hitting the legacy system. Over time, you migrate functionality until the legacy system handles nothing and can be decommissioned.

## The Proxy Layer

You need something that sits in front of both systems and routes traffic. This can be:

- A reverse proxy (nginx, HAProxy) with path-based routing
- An API gateway (Spring Cloud Gateway, Kong)
- A load balancer with URL-based rules

The simplest version:

```
/api/v2/orders    -> new Spring Boot service
/api/v1/orders    -> legacy WebLogic service
/api/*            -> legacy WebLogic service (default)
```

As you migrate each endpoint, you update the routing rules. The callers don't know (or care) which system is actually handling their request.

## Where People Get This Wrong

### 1. No Parallel Running

The temptation is to migrate an endpoint and immediately cut over. Don't. Run both implementations in parallel for a while. Route a percentage of traffic to the new service and compare results.

We ran our first migrated endpoint at 5% traffic for two weeks. Found three edge cases where the new implementation returned different results than the old one. If we'd done a full cutover, those would have been production bugs.

### 2. Shared Database (The Trap)

The legacy system and the new service both need to read and write the same data. The easy path is to point them at the same database. This works initially and becomes a nightmare later.

The legacy system assumes it owns the schema. It does raw SQL, it uses stored procedures, it depends on triggers. The moment you need to change the schema for the new service, you risk breaking the legacy system.

The better path: give the new service its own database and synchronize data. CDC (Change Data Capture) with Debezium works well here - the new service gets a real-time stream of changes from the legacy database without touching it.

### 3. Big Bang Feature Migration

Don't try to migrate an entire feature at once. Migrate individual endpoints. An "order" feature might have 15 endpoints - migrate them one at a time. It's slower, but each migration is small, testable, and reversible.

Our order of operations:
1. Read-only endpoints first (lowest risk)
2. Write endpoints that are simple (create, update)
3. Write endpoints with complex business logic (last, most carefully)

### 4. Forgetting About the Data Migration

Moving the code is half the job. Moving the data is the other half, and it's harder. The legacy database has 10 years of accumulated data, inconsistencies, and implicit business rules encoded in the data itself.

Don't try to migrate all historical data on day one. Migrate what the new service needs, keep the legacy database around for historical queries, and set a deadline for full data migration that you can push if needed.

## The Anti-Corruption Layer

The new service shouldn't know about the legacy system's data model. Build a translation layer - an anti-corruption layer in DDD terms - that converts between the legacy model and your new domain model.

```java
public class LegacyOrderAdapter {

    public Order fromLegacy(LegacyOrderRow row) {
        return Order.builder()
            .id(OrderId.of(row.getORD_NUM()))
            .customer(mapCustomer(row.getCUST_CODE()))
            .status(mapStatus(row.getSTS_FLG()))  // "A" -> ACTIVE, "C" -> CANCELLED
            .build();
    }
}
```

This layer absorbs the weirdness of the legacy model. Your new domain stays clean. When the legacy system is finally gone, you remove the adapter and nothing else changes.

## How Long Does It Take?

Longer than anyone estimates. Our first strangler fig migration was estimated at 6 months and took 14. The second was estimated at 8 months and took 10 - better, because we'd learned the pattern.

The key is that the system is functional the entire time. There's no "we'll be down for migration" period. The old system keeps running, the new system grows around it, and one day you turn off the old one and nobody notices.

That's the beauty of it. Not fast, not exciting, but it works.
