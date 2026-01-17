+++
title = "API Gateways and Spring Cloud Gateway in Practice"
date = 2026-01-17
[taxonomies]
tags = ["microservices", "spring-cloud", "spring-boot"]
+++

Every microservices architecture needs a front door. That front door is the API gateway: the single entry point that handles the cross-cutting concerns you don't want duplicated across every service. Routing, rate limiting, authentication, circuit breaking, and all the other things that feel like they should be someone else's problem.

I've used Zuul, Spring Cloud Gateway, Kong, and plain Nginx for this. Spring Cloud Gateway is what I reach for now, and it's a significant improvement over the Zuul days.

## The Evolution from Zuul

Netflix Zuul 1 was a servlet-based gateway. Blocking I/O, one thread per request. It worked fine until it didn't; under high concurrency, the thread pool would saturate and the gateway became the bottleneck. A gateway that's slower than the services behind it defeats the purpose.

Zuul 2 fixed the blocking I/O problem but never got proper Spring Cloud integration. The Spring team built Spring Cloud Gateway from scratch on top of Project Reactor and Netty. Non-blocking, reactive, and designed for high throughput.

The result: Spring Cloud Gateway handles significantly more concurrent connections with fewer resources. In our benchmarks, it handled 3x the throughput of Zuul 1 on the same hardware. For a component that sits in front of every request, that matters.

## Basic Route Configuration

Routes are the core concept. Each route maps a predicate (incoming request pattern) to a URI (downstream service) with optional filters applied along the way.

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: order-service
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
            - Method=GET,POST,PUT
          filters:
            - StripPrefix=1
            - AddRequestHeader=X-Gateway-Source, spring-cloud-gateway

        - id: product-service
          uri: lb://product-service
          predicates:
            - Path=/api/products/**
          filters:
            - StripPrefix=1

        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/api/users/**
            - Header=X-API-Version, v2
          filters:
            - StripPrefix=1
            - RewritePath=/api/users/(?<segment>.*), /v2/users/${segment}
```

The `lb://` prefix means the URI is resolved through the load balancer (Spring Cloud LoadBalancer), so it works with service discovery. Predicates can match on path, method, headers, query parameters, cookies - essentially anything in the request.

## Rate Limiting with Redis

This is a non-negotiable for any public-facing API. Without rate limiting, one misbehaving client can exhaust your resources.

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: public-api
          uri: lb://api-service
          predicates:
            - Path=/public/api/**
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 10
                redis-rate-limiter.burstCapacity: 20
                redis-rate-limiter.requestedTokens: 1
                key-resolver: "#{@apiKeyResolver}"
```

```java
@Component
public class ApiKeyResolver implements KeyResolver {
    @Override
    public Mono<String> resolve(ServerWebExchange exchange) {
        return Mono.justOrEmpty(
            exchange.getRequest().getHeaders().getFirst("X-API-Key")
        ).defaultIfEmpty("anonymous");
    }
}
```

The rate limiter uses a Redis-backed token bucket algorithm. `replenishRate` is tokens per second, `burstCapacity` is the maximum bucket size. The key resolver determines what to rate limit by - API key, IP address, user ID, whatever makes sense.

Why Redis? Because your gateway likely runs multiple instances behind a load balancer. In-memory rate limiting only works per instance, which means a client could get N times the rate limit (where N is your instance count). Redis gives you a centralized counter.

## Circuit Breaker Integration

When a downstream service is struggling, the gateway should fail fast instead of forwarding requests that will timeout.

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: payment-service
          uri: lb://payment-service
          predicates:
            - Path=/api/payments/**
          filters:
            - name: CircuitBreaker
              args:
                name: paymentCircuitBreaker
                fallbackUri: forward:/fallback/payments

resilience4j:
  circuitbreaker:
    instances:
      paymentCircuitBreaker:
        slidingWindowSize: 10
        failureRateThreshold: 50
        waitDurationInOpenState: 30s
```

```java
@RestController
@RequestMapping("/fallback")
public class FallbackController {

    @RequestMapping("/payments")
    public ResponseEntity<Map<String, String>> paymentFallback() {
        return ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE)
            .body(Map.of(
                "status", "unavailable",
                "message", "Payment service is temporarily unavailable. Please retry."
            ));
    }
}
```

The circuit breaker at the gateway level protects both the downstream service and the client. When the circuit opens, clients get an immediate 503 with a helpful message instead of waiting for a timeout.

## Timeouts

Default timeouts are almost always wrong. Set them explicitly.

```yaml
spring:
  cloud:
    gateway:
      httpclient:
        connect-timeout: 2000
        response-timeout: 5s
      routes:
        - id: report-service
          uri: lb://report-service
          predicates:
            - Path=/api/reports/**
          metadata:
            response-timeout: 30000
            connect-timeout: 2000
```

Global timeouts (5 seconds) apply to most routes. The report service gets a longer timeout (30 seconds) because report generation genuinely takes time. Be deliberate about which services get longer timeouts and document why.

## OAuth2 Integration

The gateway is the natural place to handle authentication. Validate the token once at the edge, then forward the authenticated identity to downstream services.

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://keycloak.example.com/realms/my-realm
  cloud:
    gateway:
      default-filters:
        - TokenRelay
```

```java
@Configuration
@EnableWebFluxSecurity
public class SecurityConfig {

    @Bean
    public SecurityWebFilterChain springSecurityFilterChain(ServerHttpSecurity http) {
        return http
            .authorizeExchange(exchanges -> exchanges
                .pathMatchers("/public/**").permitAll()
                .pathMatchers("/api/admin/**").hasRole("ADMIN")
                .anyExchange().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
            .csrf(ServerHttpSecurity.CsrfSpec::disable)
            .build();
    }
}
```

The `TokenRelay` filter forwards the OAuth2 token to downstream services. This means downstream services can trust the gateway's authentication and extract user information from the forwarded JWT without making their own call to the identity provider.

## API Versioning

There's no perfect way to version APIs. I've seen URL-based (`/v1/orders`, `/v2/orders`), header-based (`Accept: application/vnd.myapi.v2+json`), and query parameter-based (`?version=2`). URL-based is the most explicit and easiest to route at the gateway level.

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: order-service-v1
          uri: lb://order-service-v1
          predicates:
            - Path=/api/v1/orders/**
          filters:
            - StripPrefix=2

        - id: order-service-v2
          uri: lb://order-service-v2
          predicates:
            - Path=/api/v2/orders/**
          filters:
            - StripPrefix=2
```

The gateway routes different versions to different service instances. This lets you run V1 and V2 simultaneously, gradually migrating clients. It's also the pattern I use for canary deployments: route a percentage of traffic to the new version and monitor before cutting over fully.

## Custom Filters

When the built-in filters aren't enough, write your own. I've written custom filters for request logging, header manipulation, and response transformation.

```java
@Component
public class RequestLoggingFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String requestId = UUID.randomUUID().toString();
        ServerHttpRequest request = exchange.getRequest().mutate()
            .header("X-Request-Id", requestId)
            .build();

        log.info("Gateway request: {} {} [{}]",
            request.getMethod(), request.getURI().getPath(), requestId);

        return chain.filter(exchange.mutate().request(request).build())
            .then(Mono.fromRunnable(() ->
                log.info("Gateway response: {} [{}]",
                    exchange.getResponse().getStatusCode(), requestId)
            ));
    }

    @Override
    public int getOrder() {
        return -1; // Run early in the chain
    }
}
```

This adds a correlation ID to every request flowing through the gateway. Simple, essential for debugging, and something every API gateway should do.

## Lessons Learned

The gateway is a critical path component. Every request goes through it. That means:

1. Keep it thin. Business logic belongs in the services, not the gateway.
2. Monitor it obsessively. Latency at the gateway means latency for everyone.
3. Scale it independently. When traffic spikes, the gateway needs to handle the load before the services even see it.
4. Test your rate limiting and circuit breakers. Under load, not just with unit tests.

And one more thing: resist the temptation to put too much in the gateway. I've seen gateways that transform payloads, aggregate responses from multiple services, and implement business rules. That's not a gateway anymore; it's a BFF (Backend for Frontend) pretending to be a gateway. If you need a BFF, build a BFF. Keep the gateway focused on routing and cross-cutting concerns.
