+++
title = "Feature Flags in Spring Boot: Beyond @Profile"
date = 2026-02-14
[taxonomies]
tags = ["spring-boot", "patterns"]
+++

"Can we deploy this feature to production but keep it hidden until the product team says go?" If you've heard this question and your answer was "we'll use a Git branch," this post is for you.

Feature flags are the mechanism that lets you deploy code and activate features independently. They're essential for zero-downtime deployments, gradual rollouts, and not losing sleep the night before a release.

## Why Not @Profile?

I see this in codebases all the time:

```java
@Bean
@Profile("new-checkout")
public CheckoutService newCheckoutService() {
    return new CheckoutV2Service();
}

@Bean
@Profile("!new-checkout")
public CheckoutService oldCheckoutService() {
    return new CheckoutV1Service();
}
```

This is wrong for several reasons:

1. **Profiles are environment-level, not feature-level.** A profile applies to the entire application. You can't have "new checkout" enabled for 10% of users.
2. **Changing profiles requires a restart.** You can't toggle a feature at runtime. In a crisis, "restart the application with different profiles" is not a fast rollback.
3. **Profiles don't compose well.** When you have 5 features, you'd need 2^5 = 32 profile combinations. Nobody is managing that.
4. **No targeting.** You can't enable a feature for specific users, regions, or tenants.

Feature flags solve all of these problems.

## Rolling Your Own (The Simple Version)

For simple use cases, you don't need a library. A property-backed flag with runtime refresh works fine.

```java
@ConfigurationProperties(prefix = "app.features")
@RefreshScope
public class FeatureFlags {
    private boolean newCheckout = false;
    private boolean darkMode = false;
    private boolean betaSearch = false;

    // getters and setters
}
```

```java
@RestController
public class CheckoutController {

    private final FeatureFlags features;
    private final CheckoutV1Service v1;
    private final CheckoutV2Service v2;

    @PostMapping("/api/checkout")
    public CheckoutResult checkout(@RequestBody CheckoutRequest request) {
        if (features.isNewCheckout()) {
            return v2.process(request);
        }
        return v1.process(request);
    }
}
```

Change the property, hit `/actuator/refresh`, and the flag toggles without a restart. This works for boolean on/off flags. It doesn't work for percentage rollouts or user targeting.

## Unleash: The Open Source Option

For real feature flag management, I use Unleash. It's open source, self-hosted, and has a solid Spring Boot SDK.

```xml
<dependency>
    <groupId>io.getunleash</groupId>
    <artifactId>unleash-client-java</artifactId>
    <version>9.2.0</version>
</dependency>
```

```java
@Configuration
public class UnleashConfig {

    @Bean
    public Unleash unleash() {
        return new DefaultUnleash(UnleashConfig.builder()
            .appName("order-service")
            .instanceId("order-service-1")
            .unleashAPI("http://unleash-server:4242/api")
            .apiKey("default:development.unleash-insecure-api-token")
            .synchronousFetchOnInitialisation(true)
            .build());
    }
}
```

```java
@Service
public class CheckoutService {

    private final Unleash unleash;

    public CheckoutResult process(CheckoutRequest request, User user) {
        UnleashContext context = UnleashContext.builder()
            .userId(user.getId())
            .addProperty("country", user.getCountry())
            .addProperty("tier", user.getTier())
            .build();

        if (unleash.isEnabled("new-checkout", context)) {
            return processV2(request);
        }
        return processV1(request);
    }
}
```

Unleash supports multiple activation strategies out of the box:

- **Gradual rollout**: Enable for X% of users (by userId hash)
- **User IDs**: Enable for specific users (internal testers, beta users)
- **IPs**: Enable for specific IP ranges (office network)
- **Hostname**: Enable on specific instances
- **Custom strategies**: Write your own (enable for specific tenants, regions, etc.)

## Rollout Strategies That Work

Here's how I typically roll out a feature:

**Phase 1: Internal only.** Enable for employees using a user ID list. This catches obvious bugs before they hit real users.

**Phase 2: Canary.** Enable for 1-5% of users via gradual rollout. Monitor error rates, latency, and business metrics. If anything looks wrong, kill the flag.

**Phase 3: Ramp.** Increase to 10%, 25%, 50%. At each step, compare metrics between flag-on and flag-off cohorts. This is essentially A/B testing for free.

**Phase 4: Full rollout.** 100% of users. Keep the flag in place for a week. If something goes wrong, you can still roll back instantly.

**Phase 5: Cleanup.** Remove the flag and the old code path. This is the step everyone skips, and it's how you end up with 200 feature flags in your codebase that nobody knows the status of.

## Zero-Downtime Deployments with Flags

Feature flags decouple deployment from release. This is their superpower.

The traditional approach: develop on a branch, merge when ready, deploy, and pray. If something breaks, roll back the deployment, which means rolling back everything that was in that deployment.

The feature flag approach:

```
1. Develop behind a flag (flag OFF)
2. Merge to main and deploy (flag still OFF)
3. Enable flag for internal testing
4. Gradually roll out to users
5. If something breaks, disable flag (instant rollback)
6. Fix the issue, re-enable flag
```

The code is in production, but it's inert until the flag is on. Multiple features can be deployed simultaneously, each behind their own flag, each with independent rollout and rollback. No more "we can't deploy feature B because feature A isn't ready yet."

## Database Migrations and Flags

The tricky part. If your feature changes the database schema, you can't just toggle a flag. You need the expand/contract pattern:

**Expand**: Add new columns/tables without removing old ones. Both code paths work.

```sql
-- Expand: add new column, keep old one
ALTER TABLE orders ADD COLUMN shipping_method_v2 VARCHAR(50);
```

**Migrate**: Backfill the new column. Both code paths still work.

```java
// Feature flag controls which column the code reads from
if (features.isEnabled("new-shipping")) {
    return order.getShippingMethodV2();
}
return order.getShippingMethod();
```

**Contract**: Once the flag is fully rolled out and the old code path is removed, drop the old column.

```sql
-- Contract: remove old column (only after flag cleanup)
ALTER TABLE orders DROP COLUMN shipping_method;
ALTER TABLE orders RENAME COLUMN shipping_method_v2 TO shipping_method;
```

This is more work than a single migration, but it means you can roll back the feature without a database rollback. And database rollbacks are the scariest kind.

## Flag Hygiene

Feature flags are technical debt the moment they're created. Every flag is an `if` statement in your code, a branch in your testing matrix, and a potential source of bugs.

Rules I follow:

1. **Every flag has an owner and an expiration date.** If the flag is still in the code after the expiration date, it shows up in the team's debt backlog.
2. **Maximum flag lifetime: 30 days after full rollout.** If it's been at 100% for 30 days, remove it.
3. **Test both paths.** In CI, run tests with the flag on and off. A flag you don't test is a flag that will break.
4. **Limit the number of active flags.** More than 10-15 active flags is a code smell. Either features aren't being fully rolled out or flags aren't being cleaned up.

```java
// Track flag usage for cleanup
@Aspect
@Component
public class FeatureFlagUsageTracker {

    @Around("execution(* FeatureFlags.is*(..))")
    public Object trackUsage(ProceedingJoinPoint joinPoint) throws Throwable {
        String flagName = joinPoint.getSignature().getName();
        flagUsageMetrics.increment(flagName);
        return joinPoint.proceed();
    }
}
```

## The Operational Side

Feature flags are a production control plane. Treat them with the same seriousness as infrastructure changes.

- **Audit trail**: Log who changed which flag and when.
- **Alerts**: If a flag toggles unexpectedly, alert on it.
- **Permissions**: Not everyone should be able to toggle flags in production. The intern should not be able to enable the untested payment refactor for all users.
- **Monitoring**: When you enable a flag, watch your dashboards. Error rate, latency, business metrics. Have a rollback plan before you flip the switch.

Feature flags are simple in concept but powerful in practice. They turn risky big-bang deployments into controlled, reversible experiments. The overhead of managing them is real, but it's nothing compared to the overhead of a bad production release with no easy rollback.
