+++
title = "Service Discovery: From Eureka to Just Using Kubernetes DNS"
date = 2026-01-13
[taxonomies]
tags = ["microservices", "kubernetes", "spring-cloud"]
+++

Remember when service discovery was an entire chapter in your microservices architecture? You needed Consul or Eureka, a registration mechanism, health checks, a client-side load balancer, and a prayer that the registry stayed in sync. Those were dark times.

Kubernetes basically killed most of that complexity. But understanding *why* it killed it - and when you still need the old tools - is worth discussing.

## The Problem

In a microservices world, services come and go. Instances scale up and down. IP addresses are ephemeral. You can't hardcode `http://192.168.1.42:8080` and call it a day. You need something that maps a logical service name to actual running instances.

That's service discovery. The question is where to put that logic.

## The Old Guard: Eureka

Netflix Eureka was the go-to for Spring Cloud shops. Every service registers itself with the Eureka server on startup and sends heartbeats to prove it's still alive. Clients query Eureka to find instances and do client-side load balancing with Ribbon (later Spring Cloud LoadBalancer).

```yaml
# Eureka server
eureka:
  client:
    registerWithEureka: false
    fetchRegistry: false
  server:
    enableSelfPreservation: true

# Eureka client (every service)
eureka:
  client:
    serviceUrl:
      defaultZone: http://eureka-server:8761/eureka/
  instance:
    preferIpAddress: true
    leaseRenewalIntervalInSeconds: 10
```

```java
@SpringBootApplication
@EnableEurekaClient
public class OrderServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderServiceApplication.class, args);
    }
}
```

It worked. It also meant running and maintaining Eureka servers (clustered, because a single point of failure for service discovery is... not great), dealing with the eventual consistency of the registry, and debugging situations where a service deregistered but clients still had stale cache entries pointing at dead instances.

Self-preservation mode was especially fun. When Eureka detects a spike in registration losses, it assumes the network is the problem (not the services) and stops evicting entries. This prevents cascading failures but means your registry can contain dead instances for extended periods. There's a real tension between safety and accuracy.

## Consul: The HashiCorp Way

Consul brought a more infrastructure-level approach. Service registration, health checking, key-value store, and multi-datacenter support in one tool.

```yaml
spring:
  cloud:
    consul:
      host: consul-server
      port: 8500
      discovery:
        health-check-interval: 10s
        instance-id: ${spring.application.name}:${random.value}
```

Consul's health checks were more sophisticated than Eureka's heartbeats - you could define HTTP, TCP, script, and gRPC checks. And the gossip protocol for failure detection was faster and more reliable than Eureka's lease-based approach.

But it was still another piece of infrastructure to manage. Consul agents on every node, ACL policies, TLS certificates, cluster bootstrapping. I've spent more hours debugging Consul cluster elections than I'd like to remember.

## Kubernetes DNS: The Great Simplifier

Then Kubernetes happened, and for most teams, the service discovery problem just... went away.

When you create a Kubernetes Service, you get a stable DNS name automatically. `order-service.default.svc.cluster.local` resolves to the pods backing that service. Kubernetes handles registration (pods are registered when they pass readiness probes), deregistration (pods are removed when they fail or are terminated), and load balancing (kube-proxy distributes traffic).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: order-service
spec:
  selector:
    app: order-service
  ports:
    - port: 8080
      targetPort: 8080
```

Your Spring Boot service doesn't need any discovery client. It just calls `http://order-service:8080/api/orders` and Kubernetes handles the rest. No Eureka. No Consul. No client-side load balancer.

```java
@Bean
public RestClient orderServiceClient() {
    return RestClient.builder()
        .baseUrl("http://order-service:8080")
        .build();
}
```

That's it. The complexity that used to require dedicated infrastructure and libraries is now a platform feature.

## Spring Cloud Kubernetes

For teams that want tighter integration between Spring Cloud and Kubernetes, there's Spring Cloud Kubernetes. It reads from the Kubernetes API instead of Eureka or Consul.

```yaml
spring:
  cloud:
    kubernetes:
      discovery:
        enabled: true
        all-namespaces: false
      reload:
        enabled: true
        strategy: refresh
```

This gives you the Spring Cloud DiscoveryClient abstraction backed by the Kubernetes API. You can use `@LoadBalanced` WebClient, service instance metadata from Kubernetes labels, and config reloading from ConfigMaps and Secrets.

```java
@Bean
@LoadBalanced
public WebClient.Builder webClientBuilder() {
    return WebClient.builder();
}

// Now you can use logical service names
webClient.get()
    .uri("http://order-service/api/orders/{id}", orderId)
    .retrieve()
    .bodyToMono(Order.class);
```

Honestly? For most use cases, plain Kubernetes DNS is sufficient and you don't need this. Spring Cloud Kubernetes adds value when you need service instance metadata, dynamic config reload, or you're migrating from Eureka and want to keep the DiscoveryClient abstraction.

## When You Still Need the Old Tools

Kubernetes DNS solves service discovery inside the cluster. But there are scenarios where it's not enough:

**Multi-cluster / multi-region.** If your services span multiple Kubernetes clusters, Kubernetes DNS only works within a single cluster. Consul with its multi-datacenter support or a service mesh like Istio with multi-cluster federation fills this gap.

**Hybrid environments.** If some services run on Kubernetes and others on bare metal or VMs, you need a discovery mechanism that spans both. Consul handles this well because its agents can run anywhere.

**Advanced health checking.** Kubernetes readiness probes are binary - ready or not. If you need more nuanced health status (degraded mode, maintenance mode, weighted routing based on load), you might need something richer.

**Non-HTTP services.** Kubernetes Services work great for HTTP. For more exotic protocols or complex routing requirements, you might need a service mesh or dedicated discovery tool.

## The Service Mesh Tangent

I'm not going to go deep on service meshes here, but they deserve a mention. Istio, Linkerd, and friends add a sidecar proxy to every pod that handles service discovery, load balancing, mTLS, retries, and observability.

The pitch is compelling. The operational overhead is... less compelling. I've seen teams adopt Istio and spend more time debugging the mesh than their actual services. If you need mTLS between services and advanced traffic management, a service mesh is worth considering. If you just need service discovery, it's massive overkill.

## My Recommendation

For greenfield projects on Kubernetes: just use Kubernetes DNS. Seriously. Don't install Eureka. Don't install Consul. Just create Services and call them by name. It's built into the platform, it's well-tested, and it removes an entire category of infrastructure from your stack.

If you're migrating from Eureka, Spring Cloud Kubernetes gives you a smooth transition path without changing your service code.

If you're in a multi-cluster or hybrid environment, Consul is still the best general-purpose tool.

And if someone suggests adding a service mesh "just in case" - push back hard. You can always add it later. You can never get back the weeks you'll spend debugging Envoy sidecar injection issues.
