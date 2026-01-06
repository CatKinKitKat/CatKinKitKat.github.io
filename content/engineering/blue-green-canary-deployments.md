+++
title = "Blue-Green and Canary Deployments: How to Ship Without Sweating"
date = 2025-06-15
[taxonomies]
tags = ["kubernetes", "devops", "ci-cd"]
+++

I once deployed a database migration that was backward-incompatible with the running version of the service. The new code went live, the old pods were killed, and for about 45 seconds every request failed because the rolling update was halfway through and half the pods had the new schema while the old pods were still trying to read the old one.

That experience taught me the difference between "deploying" and "deploying safely." Rolling updates are fine for most changes. But when you need confidence, you need blue-green or canary deployments.

## Blue-Green: The Big Switch

Blue-green is conceptually the simplest. You have two identical environments: blue (current) and green (new). You deploy to green, test it, and switch traffic all at once.

On Kubernetes, the cleanest way is with two Deployments and a Service that selects one:

```yaml
# Blue deployment (current)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
      version: blue
  template:
    metadata:
      labels:
        app: order-service
        version: blue
    spec:
      containers:
        - name: order-service
          image: myregistry/order-service:1.4.2

# Service pointing to blue
apiVersion: v1
kind: Service
metadata:
  name: order-service
spec:
  selector:
    app: order-service
    version: blue  # flip this to "green" to switch
  ports:
    - port: 8080
```

Deploy the green version alongside blue. Test it directly (port-forward, internal URL, whatever). When you're confident, change the Service selector from `blue` to `green`. Traffic switches instantly.

Rollback is equally instant: flip the selector back. The blue pods are still running.

The downside: you need double the resources during the transition. For small services this is fine. For a service that needs 16 CPU cores and 32GB of RAM, running two copies gets expensive.

## Canary: The Gradual Ramp

Canary deployments send a small percentage of traffic to the new version and increase it gradually. If errors spike, you roll back. If everything looks good, you promote to 100%.

With a service mesh (Istio), this is straightforward:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: order-service
spec:
  hosts:
    - order-service
  http:
    - route:
        - destination:
            host: order-service
            subset: stable
          weight: 95
        - destination:
            host: order-service
            subset: canary
          weight: 5
```

Start at 5%. Watch metrics. Increase to 25%, 50%, 100%. At each step, check error rates, latency percentiles, and business metrics.

Without a service mesh, you can approximate canary deployments using the Kubernetes Deployment's `maxSurge` and `maxUnavailable`, but you don't get percentage-based traffic splitting. You get "some pods are new, some are old, and the load balancer distributes randomly." It works, but it's less controlled.

### Argo Rollouts

If you want canary without Istio, Argo Rollouts is the answer:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: order-service
spec:
  strategy:
    canary:
      steps:
        - setWeight: 5
        - pause: { duration: 5m }
        - setWeight: 25
        - pause: { duration: 5m }
        - setWeight: 50
        - pause: { duration: 10m }
        - setWeight: 100
      canaryService: order-service-canary
      stableService: order-service-stable
```

Argo Rollouts replaces the Deployment resource and adds canary logic with automated analysis. It integrates with Prometheus for automated rollback based on metrics.

## A/B Testing: Not the Same as Canary

Canary is about risk reduction - gradually rolling out a new version. A/B testing is about experimentation - showing different versions to different users and measuring business outcomes.

The implementation looks similar (traffic splitting), but the intent is different. With A/B testing, you typically route based on user attributes (header, cookie, user ID hash), not random percentage:

```yaml
http:
  - match:
      - headers:
          x-user-group:
            exact: "experiment-a"
    route:
      - destination:
          host: order-service
          subset: variant-a
  - route:
      - destination:
          host: order-service
          subset: variant-b
```

You need consistent routing - the same user should always see the same variant for the duration of the experiment. This is harder than it sounds when you have multiple layers of load balancing.

## The Database Problem

Here's where deployment strategies get truly painful. Your code is stateless and easy to version. Your database is stateful and shared between old and new versions during the transition.

The rule: database migrations must be backward-compatible with the previous version of the code.

This means:

**Adding a column**: fine. Old code ignores it.

**Removing a column**: do it in two releases. First release: stop using the column in code. Second release: drop the column.

**Renaming a column**: add the new column, backfill data, update code to use the new column, drop the old column. Three releases minimum.

**Changing a column type**: same multi-step dance.

Tools like Flyway and Liquibase manage the migration execution, but they don't make backward-compatible migrations automatic. You have to design them that way.

```sql
-- V1: Add new column (backward-compatible)
ALTER TABLE orders ADD COLUMN customer_email VARCHAR(255);

-- V2: Backfill data (backward-compatible)
UPDATE orders SET customer_email = (
  SELECT email FROM customers WHERE customers.id = orders.customer_id
);

-- V3: Application now reads from customer_email instead of joining

-- V4: Drop the old join dependency (only after V3 is fully deployed)
```

The expand-and-contract pattern. It's tedious but it's the only way to do zero-downtime deployments with schema changes.

## Rollback Strategies

Blue-green rollback: flip the selector. Done.

Canary rollback: set weight to 0 for canary, 100 for stable. Done.

Rolling update rollback: `kubectl rollout undo deployment/order-service`. Also done, but slower because it has to roll pods back.

Database rollback: this is the hard one. If your migration added a column, rolling back the code is fine - the column sits there unused. If your migration deleted data, you're restoring from backup.

My advice: never make destructive database changes in the same release as code changes. Separate them. Make the code change, verify it works, then make the schema cleanup in a later release.

## The Practical Setup

For most teams, I recommend:

1. Rolling updates for routine deployments (most changes are low-risk)
2. Canary via Argo Rollouts for anything touching critical paths
3. Blue-green for major version changes or infrastructure migrations
4. Expand-and-contract for all database migrations
5. Feature flags for anything that needs instant rollback without redeployment

The deployment strategy should match the risk level of the change. Not everything needs a canary. But the database migration that touches every row in your orders table? That needs every safety net you can find.
