+++
title = "Kubernetes for Java Developers: The Stuff Nobody Warns You About"
date = 2025-05-01
[taxonomies]
tags = ["kubernetes", "java", "devops"]
+++

I spent my first six months running Java on Kubernetes thinking I knew what I was doing. I had a Deployment, a Service, an Ingress. The app started. Pods were green. I declared victory.

Then things started breaking in subtle, infuriating ways. OOMKilled pods. Cascading restarts during deployments. Memory limits that made no sense. The JVM and Kubernetes have a complicated relationship, and most Java developers learn about it the hard way.

## The JVM Container Problem

Older JVMs (pre-Java 10) didn't know they were in a container. The JVM would look at the host machine's total memory, decide it could use 25% of that 64GB node for heap, and immediately get killed by the cgroup OOM killer. Kubernetes reports the pod as OOMKilled, you stare at your `-Xmx512m` flag and wonder why it's using 2GB.

Modern JVMs (17+) are container-aware by default. But you still need to be explicit:

```yaml
resources:
  requests:
    memory: "512Mi"
    cpu: "250m"
  limits:
    memory: "1Gi"
    cpu: "1000m"
```

```dockerfile
ENTRYPOINT ["java", \
  "-XX:+UseContainerSupport", \
  "-XX:MaxRAMPercentage=75.0", \
  "-XX:InitialRAMPercentage=50.0", \
  "-jar", "/app.jar"]
```

The 75% rule: give the JVM 75% of the container memory limit. The rest covers metaspace, thread stacks, native memory, and whatever else the JVM decides to allocate outside the heap. If you set `MaxRAMPercentage=90.0`, you're playing Russian roulette with the OOM killer.

## Resource Requests vs Limits: The Eternal Debate

Requests are what the scheduler uses to place your pod. Limits are the ceiling. Set requests too low and the scheduler packs too many pods on a node. Set limits too low and your pods get throttled or killed.

For Java, I've settled on this pattern: set memory request equal to memory limit. The JVM allocates a predictable amount of memory at startup and doesn't give it back. There's no point in overcommitting memory for JVM workloads - the JVM will use what it asked for.

CPU is different. Set requests to what you actually need during steady state, and limits higher to allow for bursts during startup and GC. A Spring Boot app that needs 200m normally might need 1000m during startup when it's scanning classpath annotations and initializing beans.

Some people argue you should not set CPU limits at all. Their reasoning: CPU throttling is worse than letting the pod burst. I see the point, but I've also seen a single pod eat an entire node's CPU during a GC storm and starve everything else. Set limits.

## Liveness and Readiness Probes

This is where most people mess up first:

```yaml
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 60
  periodSeconds: 10
  failureThreshold: 3
readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 5
  failureThreshold: 3
```

The distinction matters. Readiness means "I can handle traffic." Liveness means "I'm not deadlocked." If your liveness probe fails, Kubernetes kills the pod. If your readiness probe fails, the pod stays alive but traffic stops flowing to it.

Spring Boot Actuator gives you separate endpoints for each. The liveness probe checks if the app is running. The readiness probe checks if downstream dependencies (database, message broker) are available.

The mistake I see constantly: using the same endpoint for both, with aggressive timeouts. Your database has a 5-second hiccup, the liveness probe fails three times, Kubernetes kills all your pods simultaneously, and now you have a cascading outage instead of a brief slowdown.

Set `initialDelaySeconds` generously for Java apps. Spring Boot can take 30-90 seconds to start depending on how much classpath scanning it does. A startup probe is even better for this:

```yaml
startupProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  failureThreshold: 30
  periodSeconds: 5
```

This gives the app up to 150 seconds to start. Once the startup probe succeeds, the liveness and readiness probes take over.

## Graceful Shutdown

By default, when Kubernetes wants to stop a pod, it sends SIGTERM and waits 30 seconds before sending SIGKILL. Your Spring Boot app needs to handle this:

```yaml
# application.yml
server:
  shutdown: graceful
spring:
  lifecycle:
    timeout-per-shutdown-phase: 25s
```

The timeout should be less than the `terminationGracePeriodSeconds` in your pod spec (default 30s). If your app takes longer to drain connections than Kubernetes is willing to wait, connections get dropped.

The order matters too. When a pod is terminating, Kubernetes removes it from the Service endpoints. But there's a propagation delay - some traffic might still arrive after SIGTERM. Add a `preStop` hook to give the endpoints time to update:

```yaml
lifecycle:
  preStop:
    exec:
      command: ["sleep", "5"]
```

Five seconds of sleep before shutdown begins. It's ugly but it works.

## ConfigMaps and the Versioning Trick

Don't mount ConfigMaps as environment variables if you want to update them without restarting pods. Mount them as volumes - Kubernetes updates the files automatically.

But here's the real trick: version your ConfigMaps in the name and reference them from your Deployment:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config-v3
data:
  application.yml: |
    # config here
```

When you update the config, you create `myapp-config-v4` and update the Deployment reference. This triggers a rolling update, which is what you actually want - a controlled rollout, not a silent config change that might break things.

## Helm Charts: Keep Them Stupid

I've seen Helm charts that are more complex than the application they deploy. Hundreds of conditionals, deeply nested values, Go template logic that nobody can read.

Keep your Helm charts boring. A deployment, a service, a configmap, an ingress. Parameterize the things that actually change between environments (image tag, replica count, resource limits, config values). Don't parameterize everything.

```yaml
# values.yaml
replicaCount: 2
image:
  repository: myregistry.azurecr.io/myapp
  tag: "latest"
resources:
  requests:
    memory: "512Mi"
    cpu: "250m"
  limits:
    memory: "1Gi"
    cpu: "1000m"
```

If you find yourself writing complex Helm logic, you're probably doing something that should be handled by your CI/CD pipeline or a Kubernetes operator instead.

## Cluster Setup: The Stuff You'll Wish You Did First

Namespaces per team, not per environment. ResourceQuotas on every namespace. NetworkPolicies that default-deny and whitelist. PodDisruptionBudgets on anything that matters.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: myapp
```

Without a PDB, a node drain during a cluster upgrade can take down all your replicas simultaneously. Ask me how I know.

## The Honest Summary

Kubernetes is not hard for Java developers because Kubernetes is hard. It's hard because the JVM has its own opinions about memory and threads, and those opinions conflict with how Kubernetes manages resources. Understand the interaction between cgroups and the JVM, get your probes right, handle shutdown gracefully, and most of the pain goes away.

The rest is just YAML. Lots and lots of YAML.
