+++
title = "Testcontainers, or How I Learned to Love Integration Tests"
date = 2025-11-20
[taxonomies]
tags = ["testing", "spring-boot", "docker"]
+++

For years, my integration tests were a joke. They either used H2 (which behaves nothing like Postgres in production) or required a shared database that was perpetually in some unknown state because someone ran their tests thirty minutes ago and didn't clean up. Both approaches are terrible. Then Testcontainers came along and I stopped making excuses.

## The Pitch

Testcontainers spins up real Docker containers for your test dependencies. Real Postgres. Real Kafka. Real Redis. Your tests run against the same software you run in production. When the tests finish, the containers are destroyed. Every test run starts clean.

The overhead is real - starting a Postgres container takes a few seconds. But the confidence you get from testing against real infrastructure instead of mocks and in-memory fakes is worth every second.

## Database Testing That Actually Means Something

Here's the minimum setup for Spring Boot + Postgres:

```java
@SpringBootTest
@Testcontainers
class OrderRepositoryTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private OrderRepository orderRepository;

    @Test
    void shouldPersistAndRetrieveOrder() {
        Order order = new Order("ABC-123", OrderStatus.PENDING, BigDecimal.valueOf(49.99));
        orderRepository.save(order);

        Optional<Order> found = orderRepository.findByOrderNumber("ABC-123");
        assertThat(found).isPresent();
        assertThat(found.get().getStatus()).isEqualTo(OrderStatus.PENDING);
    }
}
```

This test runs against real Postgres. Your Flyway or Liquibase migrations run. Your native queries run. That `jsonb` column that H2 can't handle? Works fine. That window function in your reporting query? Actually tested now.

I've caught at least a dozen bugs that H2 would have happily ignored: Postgres-specific casting, array operations, `ON CONFLICT` clauses, and one memorable case where a query used `ILIKE` (Postgres-specific) that had been passing in H2 because H2's `LIKE` is case-insensitive by default.

## Kafka Testing

Kafka is where Testcontainers really shines, because the alternative - running a shared Kafka cluster for tests - is miserable.

```java
@Container
static KafkaContainer kafka = new KafkaContainer(
    DockerImageName.parse("confluentinc/cp-kafka:7.5.0")
);

@DynamicPropertySource
static void kafkaProperties(DynamicPropertyRegistry registry) {
    registry.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
}
```

Now you can test your producers and consumers against real Kafka. Topic creation, partition assignment, consumer group rebalancing - all real.

For tests that verify message flow end-to-end:

```java
@Test
void shouldProcessOrderMessage() {
    // Produce a message
    kafkaTemplate.send("orders", new OrderEvent("123", "CREATED")).get();

    // Wait for the consumer to process it
    await().atMost(Duration.ofSeconds(10))
        .untilAsserted(() -> {
            Order order = orderRepository.findById("123").orElseThrow();
            assertThat(order.getStatus()).isEqualTo(OrderStatus.CREATED);
        });
}
```

The `await()` from Awaitility is essential here. Kafka consumers are asynchronous; you can't just assert immediately after sending a message. I've seen people add `Thread.sleep(5000)` instead. Don't be that person.

## Spring Boot Development Mode with Testcontainers

Spring Boot 3.1 introduced Testcontainers support at development time, not just in tests. You define your containers in a test configuration and Spring Boot uses them when you run the app locally.

```java
@TestConfiguration(proxyBeanMethods = false)
public class TestcontainersConfig {

    @Bean
    @ServiceConnection
    PostgreSQLContainer<?> postgresContainer() {
        return new PostgreSQLContainer<>("postgres:16");
    }

    @Bean
    @ServiceConnection
    KafkaContainer kafkaContainer() {
        return new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.5.0"));
    }
}
```

The `@ServiceConnection` annotation is the magic. Spring Boot automatically configures the datasource and Kafka bootstrap servers from the container. No `@DynamicPropertySource` needed.

Run your app with this configuration and you've got real Postgres and real Kafka without installing anything locally except Docker. New team member onboarding goes from "spend two days setting up local infrastructure" to "have Docker installed, run the app."

## Testing Distributed Message-Driven Apps

When you have multiple services communicating through Kafka, testing the full flow gets complicated. Here's what's worked for me:

**Test each service in isolation first.** Use Testcontainers with Kafka to verify that Service A produces the right messages and Service B processes them correctly. Contract testing (Pact) verifies the message shapes match.

**For the full flow, use Docker Compose with Testcontainers:**

```java
@Container
static DockerComposeContainer<?> environment =
    new DockerComposeContainer<>(new File("src/test/resources/docker-compose-test.yml"))
        .withExposedService("postgres", 5432)
        .withExposedService("kafka", 9092)
        .withExposedService("order-service", 8080)
        .withExposedService("shipping-service", 8081)
        .waitingFor("order-service", Wait.forHttp("/actuator/health"));
```

This spins up the full ecosystem. It's slow (30+ seconds to start), so reserve it for a small number of critical path tests. Don't try to cover every edge case this way.

## Container Reuse

Starting containers for every test class adds up. Testcontainers supports reuse:

```java
static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16")
    .withReuse(true);
```

With `testcontainers.reuse.enable=true` in your `~/.testcontainers.properties`, the container persists between test runs. First run takes 5 seconds to start Postgres. Subsequent runs reuse the existing container. Huge time saver during development.

The trade-off: your test data accumulates. Use `@Sql` annotations or a `@BeforeEach` that truncates tables. Or use `@Transactional` on your test methods to roll back after each test. The container stays warm; the data stays clean.

## Testing Microservices: The Tool Belt

Testcontainers doesn't exist in isolation. Here's the combination I've landed on:

- **Unit tests** for business logic. No containers, no Spring context. Fast.
- **Testcontainers** for integration tests. Real databases, real message brokers. Medium speed.
- **Contract tests (Pact)** for API compatibility. No containers needed. Fast.
- **Gatling** for load testing. Against a deployed environment, not in unit tests. Slow but important.

The key is knowing which tool to reach for. If you're testing a SQL query, use Testcontainers. If you're testing that your JSON response matches what another service expects, use Pact. If you're testing that your service doesn't fall over at 1000 requests per second, use Gatling.

## The Gotchas

**Docker-in-Docker in CI.** If your CI runs inside Docker, you need Docker-in-Docker (DinD) or a mounted Docker socket. GitLab CI supports this natively. GitHub Actions runs on VMs, so it works out of the box. Jenkins... depends on your Jenkins setup, and it's probably going to be annoying.

**Container startup time.** A Postgres container starts in 2-3 seconds. A Kafka container takes 5-10. An Elasticsearch container takes 15-20. If you have five different container types, you're looking at 30+ seconds before your first test runs. Use container reuse in development and singleton containers in CI.

**Memory.** Each container uses real memory. If your CI runner has 4GB of RAM and you're starting Postgres, Kafka, Redis, and Elasticsearch, you might run out. Be deliberate about which tests need which containers.

Testcontainers isn't zero-cost. But compared to the cost of bugs that pass through fake database tests, it's a bargain.
