+++
title = "Load Testing with Gatling, or Breaking Things on Purpose"
date = 2025-12-01
[taxonomies]
tags = ["testing", "performance"]
+++

There are two kinds of performance problems: the ones you find during load testing and the ones that find you at 3 AM on a Friday. I prefer the first kind.

I've used JMeter, k6, Locust, and Gatling. I keep coming back to Gatling. The simulation scripts are actual code (Scala or Java), the reports are excellent out of the box, and it handles high concurrency without becoming the bottleneck itself (looking at you, JMeter with 500 threads eating all your laptop's RAM).

## Scenario Design

A load test is only as good as its scenario. If you're hitting one endpoint with a constant rate, you're testing a fantasy. Real traffic has patterns.

Here's a Gatling simulation that models something closer to reality:

```java
public class OrderFlowSimulation extends Simulation {

    HttpProtocolBuilder httpProtocol = http
        .baseUrl("https://api.staging.example.com")
        .acceptHeader("application/json")
        .contentTypeHeader("application/json");

    ScenarioBuilder browseAndOrder = scenario("Browse and Order")
        .exec(http("List Products").get("/products"))
        .pause(Duration.ofSeconds(2), Duration.ofSeconds(5))
        .exec(http("View Product").get("/products/#{productId}"))
        .pause(Duration.ofSeconds(1), Duration.ofSeconds(3))
        .exec(http("Create Order").post("/orders")
            .body(StringBody("{\"productId\": \"#{productId}\", \"quantity\": 1}"))
            .check(jsonPath("$.orderId").saveAs("orderId")))
        .pause(Duration.ofMillis(500))
        .exec(http("Check Order Status").get("/orders/#{orderId}"));

    ScenarioBuilder justBrowsing = scenario("Just Browsing")
        .exec(http("List Products").get("/products"))
        .pause(Duration.ofSeconds(1), Duration.ofSeconds(10))
        .exec(http("View Product").get("/products/#{productId}"));

    {
        setUp(
            browseAndOrder.injectOpen(
                rampUsersPerSec(1).to(20).during(Duration.ofMinutes(2)),
                constantUsersPerSec(20).during(Duration.ofMinutes(5)),
                rampUsersPerSec(20).to(50).during(Duration.ofMinutes(2))
            ),
            justBrowsing.injectOpen(
                constantUsersPerSec(100).during(Duration.ofMinutes(9))
            )
        ).protocols(httpProtocol);
    }
}
```

Key points: multiple scenarios with different user behaviors, think time between requests (the `pause` calls), and a ramp-up period. Slamming 1000 users at your service from second zero tells you nothing useful. Real traffic builds up gradually.

## Breaking Things on Purpose

The whole point of load testing is to find the breaking point *before* production does. I structure my tests in three phases:

**Phase 1: Baseline.** Low load, confirm everything works. 10 requests per second for 5 minutes. If anything fails here, you have a functional bug, not a performance bug.

**Phase 2: Expected load.** The traffic you anticipate in production. If your service handles 500 requests per second in production, test at 500 rps for 15 minutes. Check that response times stay within your SLOs and error rates stay at zero.

**Phase 3: Break it.** Ramp up until things start failing. Keep going. Find out what fails first: the application (thread pool exhaustion, OOM), the database (connection pool, query timeouts), the network (bandwidth, connection limits), or something you didn't think of (that one downstream service that can't handle the load).

The third phase is where the valuable information is. Knowing your service breaks at 2000 rps because the database connection pool maxes out is actionable. Knowing it "works fine at 500 rps" is nice but incomplete.

## Interpreting Results

Gatling's HTML reports are good. But you need to know what to look at.

**Don't focus on average response time.** Averages lie. A service with a 50ms average could have a p99 of 5 seconds. The average hides the outliers, and outliers are what your users complain about.

**Focus on percentiles.** p50 tells you the median experience. p95 tells you what 1 in 20 users experience. p99 tells you what your worst users see. If your p99 is 10x your p50, you have a tail latency problem.

**Watch for response time degradation.** A healthy system has flat response times under increasing load until it hits capacity. Then response times spike. If response times increase linearly with load from the start, you have a contention issue (probably database locks or thread pool saturation).

**Error rate matters more than response time.** A slow response is bad. A failed response is worse. If your error rate goes from 0% to 5% at 1000 rps, that's 50 failed requests per second. Depending on your business, each of those could be a lost order.

## Performance Regression Testing

Here's where most teams stop: they run a load test once, pat themselves on the back, and never run it again. Then six months later, someone adds a new database query in a hot path and response times double.

Performance regression testing means running your load tests in CI. Not the full break-it scenario - that takes too long and needs a dedicated environment. But a scaled-down version that catches regressions:

```java
{
    setUp(
        criticalPath.injectOpen(
            constantUsersPerSec(50).during(Duration.ofMinutes(2))
        )
    ).protocols(httpProtocol)
     .assertions(
         global().responseTime().percentile3().lt(500),
         global().responseTime().percentile4().lt(1000),
         global().successfulRequests().percent().gt(99.0)
     );
}
```

The `assertions` block is the key. If p95 exceeds 500ms or p99 exceeds 1000ms, the test fails. The build breaks. Someone investigates.

This won't catch every performance issue - a 2-minute test at 50 rps won't reveal problems that only appear under sustained high load. But it catches the obvious regressions: N+1 queries, missing indexes, accidental eager fetching of entire tables.

## High-Performance Load Testing

When you need to generate serious load (10k+ rps), a single Gatling instance on your laptop won't cut it. Options:

**Gatling Enterprise** (the paid version) handles distributed load generation. Multiple injectors coordinated from a central controller, with merged reporting.

**DIY with Kubernetes.** Run multiple Gatling instances as Kubernetes Jobs. Each runs a portion of the load. You merge the results afterward. It works, but the reporting merge is manual and annoying.

**Right-size your test.** Before reaching for distributed load generation, ask whether you actually need 10k rps in your test. If your production traffic is 500 rps, testing at 2000 rps (4x) is probably enough to find the bottlenecks. You don't always need to simulate Black Friday.

## The Non-Obvious Stuff

**Test your database migrations under load.** Run your Flyway migration while the service is handling traffic. I've seen zero-downtime deployments fail because a Postgres `ALTER TABLE` took a lock that blocked all queries for 30 seconds.

**Test with realistic data volumes.** An empty database is fast. A database with 10 million rows behaves differently. Seed your test environment with production-scale data before running load tests.

**Test the unhappy paths.** What happens when the downstream service returns errors? Does your service degrade gracefully or does it cascade? Combine load testing with chaos engineering - kill a dependency mid-test and see what happens.

**Monitor during the test.** Gatling shows you response times and error rates. But the root cause is in your metrics: CPU usage, heap pressure, GC pauses, connection pool utilization, database query times. Run Gatling while watching Grafana.

Load testing isn't something you do once and forget. It's a practice. Build it into your pipeline, run it regularly, and treat performance regressions with the same urgency as functional bugs. Your 3 AM self will thank you.
