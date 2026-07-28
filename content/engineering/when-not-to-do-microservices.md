+++
title = "When NOT to Do Microservices"
date = 2026-01-02
[taxonomies]
tags = ["microservices", "architecture", "opinion"]
+++

I'm going to say something controversial for a backend developer at a consultancy: most projects I've seen adopt microservices shouldn't have. There, I said it. Let me explain before the architecture astronauts come for me.

## The Default Should Be a Monolith

Somewhere along the way, "monolith" became a dirty word. Like admitting you deploy a single artifact makes you a lesser engineer. That's nonsense. A well-structured monolith is faster to develop, easier to debug, simpler to deploy, and cheaper to run than a poorly-designed microservices architecture. And most microservices architectures I've encountered in the wild are poorly designed.

The question shouldn't be "should we use microservices?" It should be "do we have a specific problem that microservices solve better than a monolith?" If you can't answer that clearly, you don't need them.

## The Data Problem

This is the one that kills most microservices adoptions. In a monolith, your data lives in one database. You can join tables. You can have foreign keys. Transactions are ACID. Life is straightforward.

The moment you split into services, each with its own database (because that's what the books say), you've traded SQL joins for network calls, foreign keys for eventual consistency, and ACID transactions for distributed sagas that make you question your career choices.

```java
// Monolith: simple, correct, fast
SELECT o.*, c.name, p.title
FROM orders o
JOIN customers c ON o.customer_id = c.id
JOIN products p ON o.product_id = p.id
WHERE o.id = ?

// Microservices: three network calls, eventual consistency,
// error handling for each call, and what happens when
// the product service is down?
Order order = orderService.getOrder(id);
Customer customer = customerClient.getCustomer(order.getCustomerId());
Product product = productClient.getProduct(order.getProductId());
```

I've watched teams spend months building saga orchestrators to replicate what a database transaction gives you for free. That's not engineering. That's self-inflicted complexity.

## The "Calling Your Services" Problem

In theory, microservices are independently deployable. In practice, Service A calls Service B calls Service C, and if you change the contract on C, you have to coordinate deployments across all three. You've replaced compile-time dependencies with runtime dependencies, which is strictly worse because now they fail at 3 AM instead of during the build.

The distributed monolith is what happens when you split by technical layer instead of business capability. I've seen it more times than I can count:

- A "user service" that every other service depends on
- A "notification service" that has to know about every domain event
- A "common library" shared across all services that gets updated every sprint

Congratulations, you've built a monolith with network latency.

## Carve by Verticals, Not Layers

If you absolutely must go microservices - and sometimes you genuinely must - carve by business verticals, not technical layers. A good microservice owns a business capability end-to-end: its own data, its own UI if applicable, its own deployment pipeline.

Bad boundaries:
- User Service, Notification Service, Logging Service, Auth Service
- Basically anything that's a horizontal concern

Better boundaries:
- Order Management (handles the full lifecycle of an order)
- Inventory (owns stock levels and warehouse integration)
- Billing (owns invoices, payments, and reconciliation)

Each of these can function independently. If the billing service goes down, customers can still browse products and place orders (you just process the payment later). That's real independence, not the fake kind where your "independent" services share a database and can't start without each other.

## The Team Argument

The one genuinely good reason for microservices is organizational. If you have multiple autonomous teams that need to deploy independently on different cadences, microservices give you that. Conway's Law is real - your architecture will mirror your communication structure whether you plan for it or not.

But here's the thing: most companies I consult for don't have that problem. They have one team, maybe two, and they're all deploying on the same two-week sprint cycle. Microservices add overhead without any organizational benefit.

## The Real Checklist

Before going microservices, honestly answer these:

1. **Do you have multiple teams that need independent deployment?** Not "will we someday" - do you right now?
2. **Do you have genuinely different scaling requirements?** As in, one component needs 50x the resources of another. Not "it might scale differently someday."
3. **Can you afford the operational overhead?** Distributed tracing, service mesh, container orchestration, independent CI/CD pipelines - this isn't free.
4. **Do you understand your domain boundaries?** If you haven't built the system as a monolith first, you're guessing where to split. You'll guess wrong.

If you answered "no" to any of these, start with a modular monolith. You can always extract services later when you understand the domain better. Going the other direction - merging microservices back into a monolith - is significantly harder.

## The Modular Monolith Compromise

This is what I actually recommend for most projects. A single deployable artifact, but with strict module boundaries enforced at the code level. Spring Modulith does this well (more on that in a future post). You get the organizational clarity of service boundaries without the operational complexity of a distributed system.

When a module genuinely needs to be extracted - because of scaling, team autonomy, or technology requirements - you extract it. The module boundaries are already defined. The interfaces are already clean. The extraction is surgical instead of exploratory.

## My Unpopular Opinion

The microservices movement set the industry back by convincing a generation of developers that distributed systems are the default architecture. They're not. They're a tool for specific problems. Use them when those problems exist, not because a conference speaker told you monoliths are bad.

The best architecture is the simplest one that solves your actual problems. For most teams, that's a monolith. And there's absolutely nothing wrong with that.
