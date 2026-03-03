+++
title = "Spring Cloud Gateway: Beyond the Getting Started Guide"
date = 2026-03-03
[taxonomies]
tags = ["spring-cloud", "microservices", "spring-boot"]
+++

If you've read the Spring Cloud Gateway documentation, you know how to define routes and apply filters. Congratulations, you've covered about 20% of what you'll actually need in production. The other 80% is rate limiting that actually works, circuit breakers that don't surprise you, OAuth2 integration that doesn't make you cry, and load balancing that handles real-world scenarios.

Let me cover that 80%.

## Route Configuration: The Java DSL

YAML configuration works for simple setups. For anything dynamic or conditional, the Java DSL is cleaner.

```java
@Configuration
public class GatewayRouteConfig {

    @Bean
    public RouteLocator customRoutes(RouteLocatorBuilder builder) {
        return builder.routes()
            .route("order-service", r -> r
                .path("/api/orders/**")
                .and().method("GET", "POST", "PUT")
                .filters(f -> f
                    .stripPrefix(1)
                    .addRequestHeader("X-Gateway-Routed", "true")
                    .retry(config -> config
                        .setRetries(2)
                        .setStatuses(HttpStatus.SERVICE_UNAVAILABLE)
                        .setBackoff(Duration.ofMillis(100), Duration.ofMillis(500), 2, true))
                )
                .uri("lb://order-service"))

            .route("product-search", r -> r
                .path("/api/search/**")
                .filters(f -> f
                    .stripPrefix(1)
                    .circuitBreaker(config -> config
                        .setName("searchCircuitBreaker")
                        .setFallbackUri("forward:/fallback/search"))
                    .requestRateLimiter(config -> config
                        .setRateLimiter(redisRateLimiter())
                        .setKeyResolver(userKeyResolver()))
                )
                .uri("lb://search-service"))

            .route("legacy-redirect", r -> r
                .path("/old-api/**")
                .filters(f -> f
                    .rewritePath("/old-api/(?<segment>.*)", "/api/v2/${segment}")
                    .setStatus(HttpStatus.MOVED_PERMANENTLY))
                .uri("lb://api-service"))

            .build();
    }
}
```

The Java DSL makes it easy to compose complex routing logic and reuse filter configurations. For routes that change at runtime (A/B testing, canary deployments), you can implement `RouteLocator` backed by a database or config service.

## Rate Limiting with Redis: Production Configuration

The basic `RequestRateLimiter` filter works, but production needs are more nuanced. Different endpoints need different limits. Different clients need different limits. And you need to handle Redis being unavailable without killing all traffic.

```java
@Bean
public RedisRateLimiter redisRateLimiter() {
    // Default: 50 requests/second, burst up to 100
    return new RedisRateLimiter(50, 100, 1);
}

@Bean
public KeyResolver userKeyResolver() {
    return exchange -> {
        // Rate limit by API key first, fall back to IP
        String apiKey = exchange.getRequest().getHeaders().getFirst("X-API-Key");
        if (apiKey != null) {
            return Mono.just("api:" + apiKey);
        }

        String ip = Optional.ofNullable(exchange.getRequest().getHeaders().getFirst("X-Forwarded-For"))
            .map(xff -> xff.split(",")[0].trim())
            .orElse(exchange.getRequest().getRemoteAddress().getAddress().getHostAddress());
        return Mono.just("ip:" + ip);
    };
}
```

For different rate limits per route, configure them inline:

```java
.filters(f -> f
    .requestRateLimiter(config -> config
        .setRateLimiter(new RedisRateLimiter(10, 20, 1))  // Stricter limit
        .setKeyResolver(userKeyResolver())
        .setDenyEmptyKey(false)
        .setStatusCode(HttpStatus.TOO_MANY_REQUESTS))
)
```

The Redis rate limiter uses a Lua script for atomicity. It's solid. But what happens when Redis goes down? By default, the gateway rejects all requests. That's probably not what you want.

```java
@Bean
public RedisRateLimiter resilientRateLimiter() {
    RedisRateLimiter limiter = new RedisRateLimiter(50, 100, 1);
    // When Redis is down, allow requests through (fail open)
    limiter.setDenyEmptyKey(false);
    return limiter;
}
```

Fail open or fail closed is a deliberate decision. For public APIs, I fail open because blocking all traffic is worse than temporarily having no rate limits. For internal APIs with known abusive clients, I might fail closed.

## Circuit Breaker Integration: Getting It Right

The circuit breaker filter delegates to Resilience4j. The configuration lives in two places: the route filter definition and the Resilience4j properties.

```java
.filters(f -> f
    .circuitBreaker(config -> config
        .setName("paymentCircuitBreaker")
        .setFallbackUri("forward:/fallback/payment")
        .setRouteId("payment-service")
        .addStatusCode("500")
        .addStatusCode("503"))
)
```

```yaml
resilience4j:
  circuitbreaker:
    instances:
      paymentCircuitBreaker:
        slidingWindowType: TIME_BASED
        slidingWindowSize: 60
        minimumNumberOfCalls: 10
        failureRateThreshold: 50
        waitDurationInOpenState: 30s
        permittedNumberOfCallsInHalfOpenState: 5
        recordExceptions:
          - org.springframework.web.client.HttpServerErrorException
          - java.io.IOException
  timelimiter:
    instances:
      paymentCircuitBreaker:
        timeoutDuration: 5s
```

The time-based sliding window (60 seconds) works better than count-based for gateways because traffic volume varies throughout the day. Count-based windows can trigger too aggressively during low-traffic periods.

The time limiter wraps each call with a timeout. If the downstream service doesn't respond within 5 seconds, the call is cancelled and counted as a failure. Without this, slow responses don't trigger the circuit breaker - they just pile up.

## Fallback Controllers

Your fallback responses should be useful, not just "something went wrong."

```java
@RestController
@RequestMapping("/fallback")
public class GatewayFallbackController {

    @RequestMapping("/payment")
    public ResponseEntity<ApiError> paymentFallback(ServerWebExchange exchange) {
        String path = exchange.getRequest().getPath().toString();
        return ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE)
            .body(new ApiError(
                "PAYMENT_SERVICE_UNAVAILABLE",
                "The payment service is temporarily unavailable. Your request has not been processed. Please retry in a few minutes.",
                path,
                Instant.now()
            ));
    }

    @RequestMapping("/search")
    public ResponseEntity<SearchFallback> searchFallback() {
        // Return cached/default results instead of an error
        return ResponseEntity.ok(new SearchFallback(
            List.of(), // Empty results
            "Search is temporarily unavailable. Showing limited results.",
            true // Flag for the frontend to show a warning
        ));
    }
}
```

For non-critical paths (search, recommendations), returning a degraded response is better than an error. For critical paths (payments), an explicit error with a "please retry" message is more honest.

## OAuth2: Token Relay and Validation

The gateway can serve as the authentication boundary. Validate tokens at the edge, relay them to downstream services.

```java
@Configuration
@EnableWebFluxSecurity
public class GatewaySecurityConfig {

    @Bean
    public SecurityWebFilterChain securityFilterChain(ServerHttpSecurity http) {
        return http
            .authorizeExchange(auth -> auth
                .pathMatchers("/api/public/**").permitAll()
                .pathMatchers("/actuator/health/**").permitAll()
                .pathMatchers("/fallback/**").permitAll()
                .pathMatchers("/api/admin/**").hasAuthority("SCOPE_admin")
                .anyExchange().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt.jwtAuthenticationConverter(grantedAuthoritiesExtractor()))
            )
            .csrf(ServerHttpSecurity.CsrfSpec::disable) // API gateway, not a web app
            .build();
    }

    private Converter<Jwt, Mono<AbstractAuthenticationToken>> grantedAuthoritiesExtractor() {
        JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(jwt -> {
            List<String> roles = jwt.getClaimAsStringList("roles");
            if (roles == null) return List.of();
            return roles.stream()
                .map(role -> new SimpleGrantedAuthority("ROLE_" + role))
                .collect(Collectors.toList());
        });
        return new ReactiveJwtAuthenticationConverterAdapter(converter);
    }
}
```

Add the `TokenRelay` filter to forward the validated JWT to downstream services:

```java
.route("order-service", r -> r
    .path("/api/orders/**")
    .filters(f -> f
        .stripPrefix(1)
        .tokenRelay())
    .uri("lb://order-service"))
```

The downstream service receives the JWT in the `Authorization` header and can extract claims without re-validating against the IdP. Trust the gateway.

## Load Balancing

Spring Cloud Gateway uses Spring Cloud LoadBalancer for `lb://` URIs. The default is round-robin, which is fine for most cases. For more sophisticated strategies:

```java
@Configuration
public class LoadBalancerConfig {

    @Bean
    public ReactorLoadBalancer<ServiceInstance> orderServiceLoadBalancer(
            ServiceInstanceListSupplier supplier) {
        return new RandomLoadBalancer(supplier, "order-service");
    }
}
```

For health-aware load balancing:

```yaml
spring:
  cloud:
    loadbalancer:
      health-check:
        initial-delay: 5s
        interval: 10s
      configurations: health-check
```

This periodically checks instance health and removes unhealthy instances from the rotation. Combined with Kubernetes readiness probes, this provides two layers of health checking.

## Request and Response Modification

Real-world gateway needs often involve modifying requests and responses. Adding headers, rewriting paths, transforming bodies.

```java
@Component
public class AddCorrelationIdFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String correlationId = exchange.getRequest().getHeaders()
            .getFirst("X-Correlation-Id");

        if (correlationId == null) {
            correlationId = UUID.randomUUID().toString();
        }

        ServerHttpRequest request = exchange.getRequest().mutate()
            .header("X-Correlation-Id", correlationId)
            .build();

        String finalCorrelationId = correlationId;
        return chain.filter(exchange.mutate().request(request).build())
            .then(Mono.fromRunnable(() ->
                exchange.getResponse().getHeaders()
                    .add("X-Correlation-Id", finalCorrelationId)
            ));
    }

    @Override
    public int getOrder() {
        return Ordered.HIGHEST_PRECEDENCE;
    }
}
```

This ensures every request flowing through the gateway has a correlation ID, whether the client provided one or not. The same ID appears in the response, making it easy to trace a request through the entire system.

## Monitoring and Observability

A gateway without monitoring is a liability. You need to know latency per route, error rates, circuit breaker states, and rate limiter activity.

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,gateway,metrics,circuitbreakers
  metrics:
    tags:
      application: api-gateway
    distribution:
      percentiles-histogram:
        spring.cloud.gateway.requests: true
```

The `spring.cloud.gateway.requests` metric gives you timing data per route. Feed this into Prometheus/Grafana, set up alerts for latency spikes, and you'll catch problems before your users do.

Spring Cloud Gateway in production is not complicated, but it demands attention to the details that tutorials skip. Rate limiting, circuit breaking, authentication, and observability aren't optional features - they're what make the gateway worth having in the first place.
