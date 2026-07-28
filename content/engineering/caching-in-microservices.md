+++
title = "Caching in Microservices: When It Helps and When It Hurts"
date = 2026-01-29
[taxonomies]
tags = ["microservices", "performance", "redis"]
+++

Caching is the first thing people reach for when performance is bad and the last thing they think about when debugging stale data issues. I've been on both sides. The trick isn't knowing how to cache - it's knowing *when* to cache, and more importantly, when to stop.

## The Spring Boot Cache Abstraction

Spring Boot makes caching almost too easy. Add the annotation, pick a provider, and you're done. That ease of use is both a blessing and a trap.

```java
@Configuration
@EnableCaching
public class CacheConfig {
    // That's it. Spring Boot auto-configures the cache manager.
}

@Service
public class ProductService {

    @Cacheable(value = "products", key = "#productId")
    public Product getProduct(String productId) {
        log.info("Cache miss for product {}", productId);
        return productRepository.findById(productId)
            .orElseThrow(() -> new ProductNotFoundException(productId));
    }

    @CachePut(value = "products", key = "#product.id")
    public Product updateProduct(Product product) {
        return productRepository.save(product);
    }

    @CacheEvict(value = "products", key = "#productId")
    public void deleteProduct(String productId) {
        productRepository.deleteById(productId);
    }
}
```

Simple. Slap `@Cacheable` on a method, and Spring intercepts the call, checks the cache, and returns the cached value if it exists. Cache miss? Execute the method and store the result.

But this simplicity hides important questions. Where is the cache? How big can it grow? When do entries expire? What happens when another instance updates the product?

## In-Memory Caching (Caffeine)

For single-instance services or data that's okay being slightly stale per-instance, Caffeine is fast and simple.

```yaml
spring:
  cache:
    type: caffeine
    caffeine:
      spec: maximumSize=10000,expireAfterWrite=5m
```

Caffeine is absurdly fast. It's a near-optimal cache implementation with O(1) operations and smart eviction (W-TinyLFU). For read-heavy workloads where you can tolerate instance-level inconsistency, it's the right choice.

The problem: each instance has its own cache. Instance A caches product X, instance B updates product X, and instance A serves stale data until the TTL expires. For some data this is fine. For pricing data? Not so much.

## Redis: The Distributed Answer

When you need a shared cache across instances, Redis is the default choice. And for good reason - it's fast, it's well-understood, and the Spring Boot integration is solid.

```yaml
spring:
  data:
    redis:
      host: redis-cluster
      port: 6379
      timeout: 2s
  cache:
    type: redis
    redis:
      time-to-live: 600000   # 10 minutes
      cache-null-values: false
      key-prefix: "myapp:"
```

```java
@Configuration
@EnableCaching
public class RedisCacheConfig {

    @Bean
    public RedisCacheConfiguration cacheConfiguration() {
        return RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10))
            .serializeKeysWith(
                RedisSerializationContext.SerializationPair
                    .fromSerializer(new StringRedisSerializer()))
            .serializeValuesWith(
                RedisSerializationContext.SerializationPair
                    .fromSerializer(new GenericJackson2JsonRedisSerializer()));
    }

    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory factory) {
        Map<String, RedisCacheConfiguration> configs = Map.of(
            "products", cacheConfiguration().entryTtl(Duration.ofMinutes(30)),
            "users", cacheConfiguration().entryTtl(Duration.ofMinutes(5)),
            "sessions", cacheConfiguration().entryTtl(Duration.ofHours(1))
        );

        return RedisCacheManager.builder(factory)
            .cacheDefaults(cacheConfiguration())
            .withInitialCacheConfigurations(configs)
            .build();
    }
}
```

Different caches get different TTLs because not all data has the same staleness tolerance. Product catalogs change infrequently - 30 minutes is fine. User profiles change more often - 5 minutes. Sessions need to last - 1 hour.

## Hazelcast and In-Memory Data Grids

Sometimes Redis isn't enough. If you need distributed data structures, near-cache (a local cache backed by a distributed one), or event-driven invalidation, an in-memory data grid like Hazelcast fills that gap.

```java
@Configuration
public class HazelcastConfig {

    @Bean
    public Config hazelcastConfig() {
        Config config = new Config();
        config.setClusterName("my-application");

        MapConfig productCache = new MapConfig("products")
            .setTimeToLiveSeconds(600)
            .setMaxIdleSeconds(300)
            .setEvictionConfig(new EvictionConfig()
                .setSize(10000)
                .setEvictionPolicy(EvictionPolicy.LFU))
            .setNearCacheConfig(new NearCacheConfig()
                .setTimeToLiveSeconds(60)
                .setMaxIdleSeconds(30));

        config.addMapConfig(productCache);
        return config;
    }
}
```

The near-cache feature is Hazelcast's killer feature for read-heavy workloads. You get the speed of a local cache with the consistency of a distributed one. When an entry is updated in the cluster, the near-cache is invalidated automatically.

That said, I default to Redis unless I have a specific reason to need Hazelcast. Redis is simpler to operate and better understood by most teams.

## JPA Second-Level Cache

Hibernate has its own cache layer. The first-level cache is the persistence context (one per transaction, automatic). The second-level cache is shared across sessions and optional.

```yaml
spring:
  jpa:
    properties:
      hibernate:
        cache:
          use_second_level_cache: true
          region.factory_class: org.hibernate.cache.jcache.JCacheRegionFactory
        javax:
          cache:
            provider: com.github.benmanes.caffeine.jcache.spi.CaffeineCachingProvider
```

```java
@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Product {
    @Id
    private String id;
    private String name;
    private BigDecimal price;

    @Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
    @OneToMany(mappedBy = "product")
    private List<ProductVariant> variants;
}
```

I'm conflicted about the second-level cache. It can dramatically reduce database load for read-heavy entities. But it adds a layer of magic that makes debugging harder. When you're staring at a stale price wondering where it's coming from, the Hibernate L2 cache is often the culprit.

My rule: use it for reference data (countries, currencies, categories) that changes rarely. Don't use it for transactional data that multiple services write to.

## Cache Synchronization Across Services

This is where caching in microservices gets genuinely hard. Service A caches data from Service B. Service B updates the data. Service A still serves the old version.

Options, from simple to complex:

**TTL-based expiry.** Accept staleness. Set a reasonable TTL and move on. This is the right answer more often than people think.

**Event-driven invalidation.** Service B publishes an event when data changes. Service A listens and invalidates its cache.

```java
@KafkaListener(topics = "product-updates")
public void onProductUpdate(ProductUpdatedEvent event) {
    cacheManager.getCache("products").evict(event.getProductId());
    log.info("Evicted product {} from cache", event.getProductId());
}
```

**Cache-aside with version checks.** Include a version or last-modified timestamp. On cache hit, check the version against the source of truth.

There's no free lunch. TTL is simple but stale. Events are fresher but add infrastructure complexity. Version checks are accurate but add latency.

## Request-Level Memoization

Sometimes you don't need a persistent cache. You just need to avoid calling the same service ten times within a single request.

```java
@Component
@RequestScope
public class RequestScopedCache {
    private final Map<String, Object> cache = new HashMap<>();

    @SuppressWarnings("unchecked")
    public <T> T computeIfAbsent(String key, Supplier<T> supplier) {
        return (T) cache.computeIfAbsent(key, k -> supplier.get());
    }
}

@Service
public class OrderEnrichmentService {

    private final RequestScopedCache requestCache;

    public EnrichedOrder enrich(Order order) {
        // Called once per request, even if multiple order lines
        // reference the same customer
        Customer customer = requestCache.computeIfAbsent(
            "customer:" + order.getCustomerId(),
            () -> customerClient.getCustomer(order.getCustomerId())
        );
        // ...
    }
}
```

This is lightweight, has no persistence concerns, and perfectly solves the "N+1 API call" problem within a single request. I use it all the time.

## When Caching Hurts

Here's the part nobody talks about at conferences.

**Cache invalidation bugs are insidious.** The system works perfectly 99% of the time. Then a customer sees a stale price, or an old address, or a deleted product that's still showing. These bugs are hard to reproduce and harder to debug.

**Cache stampede.** A popular cache entry expires. Suddenly, 1000 concurrent requests all hit the database at once because they all got a cache miss simultaneously. Use cache locking or probabilistic early recomputation to prevent this.

```java
// Cache stampede protection with Caffeine
LoadingCache<String, Product> cache = Caffeine.newBuilder()
    .maximumSize(10000)
    .refreshAfterWrite(Duration.ofMinutes(4))   // Refresh in background
    .expireAfterWrite(Duration.ofMinutes(5))     // Hard expiry
    .build(key -> productRepository.findById(key).orElse(null));
```

**Caching masks real problems.** The database query takes 3 seconds? Just cache it! Now nobody notices (or fixes) the missing index until the cache goes down and suddenly everything is slow. I've seen this pattern repeatedly. Fix the root cause first. Cache second.

**Memory pressure.** Unbounded caches grow until they crash your application. Always set maximum sizes. Always monitor cache hit rates and memory usage. A cache with a 10% hit rate is just wasting memory.

## My Caching Decision Framework

1. Measure first. Is caching even necessary? Profile the actual latency.
2. Fix the root cause. Slow query? Add an index. Slow API? Optimize the endpoint.
3. Start with request-level memoization. Zero infrastructure, zero staleness issues.
4. Graduate to Caffeine for local, single-instance caching.
5. Move to Redis when you need shared cache across instances.
6. Add event-driven invalidation only when TTL-based expiry isn't acceptable.
7. Monitor hit rates. A cache that isn't hit isn't helping.

Caching is a tool, not a default. Use it deliberately, measure its impact, and always know your invalidation strategy before you write the first `@Cacheable`.
