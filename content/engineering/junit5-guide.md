+++
title = "JUnit 5 - Everything You Actually Need"
date = 2026-03-31
[taxonomies]
tags = ["java", "testing"]
+++

I've been writing JUnit 5 tests since it was called JUnit Jupiter and nobody was sure if it would stick. It stuck. Here's everything I've learned about using it effectively, minus the stuff you'll never need.

## Architecture: Why It Matters

JUnit 5 isn't a single library. It's three:

- **JUnit Platform** - the foundation for launching testing frameworks on the JVM
- **JUnit Jupiter** - the new programming model and extension model (this is what you write tests with)
- **JUnit Vintage** - backward compatibility with JUnit 4 tests

The platform/engine split means your test runner is decoupled from the test framework. You can run Jupiter tests, Vintage tests, and even custom engines (like Kotest for Kotlin) side by side. In practice, you add `junit-jupiter` as a dependency and move on. But understanding the architecture helps when things go wrong - and they will go wrong, usually because of conflicting JUnit Platform versions in the classpath.

```xml
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>5.11.0</version>
    <scope>test</scope>
</dependency>
```

## The Basics, Quickly

```java
@Test
@DisplayName("Creating an order with no items should throw")
void emptyOrderShouldThrow() {
    assertThrows(IllegalArgumentException.class, () -> new Order(List.of()));
}
```

`@DisplayName` is optional but I use it everywhere. Test method names are for code, display names are for humans reading test reports. `void emptyOrderShouldThrow` is fine for a developer; "Creating an order with no items should throw" is better in a CI report.

## Lifecycle: @BeforeEach vs @BeforeAll

```java
@BeforeAll   // Runs once before all tests in the class (static by default)
@BeforeEach  // Runs before each test
@AfterEach   // Runs after each test
@AfterAll    // Runs once after all tests in the class (static by default)
```

By default, JUnit creates a new instance of the test class for each test method. This means `@BeforeAll` and `@AfterAll` must be static. If you hate that, change the lifecycle:

```java
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class OrderServiceTest {
    @BeforeAll
    void setup() { /* no longer needs to be static */ }
}
```

I use `PER_CLASS` when I have expensive setup (like starting an embedded database) that I want to share across tests. The trade-off: tests can leak state into each other. Be disciplined about cleanup.

## Parameterized Tests: Stop Copy-Pasting

This is the feature that saves me the most time.

```java
@ParameterizedTest
@ValueSource(strings = {"", " ", "\t", "\n"})
void blankStringShouldBeRejected(String input) {
    assertThrows(IllegalArgumentException.class, () -> new Email(input));
}
```

For more complex scenarios, `@MethodSource`:

```java
@ParameterizedTest
@MethodSource("orderScenarios")
void shouldCalculateTotal(List<LineItem> items, BigDecimal expectedTotal) {
    Order order = new Order(items);
    assertEquals(expectedTotal, order.total());
}

static Stream<Arguments> orderScenarios() {
    return Stream.of(
        Arguments.of(List.of(new LineItem("A", 2, TEN)), new BigDecimal("20")),
        Arguments.of(List.of(new LineItem("B", 1, ONE)), ONE),
        Arguments.of(List.of(), BigDecimal.ZERO)
    );
}
```

`@CsvSource` is great for simple cases:

```java
@ParameterizedTest
@CsvSource({
    "100, 10, 90",
    "50, 50, 0",
    "200, 0, 200"
})
void shouldApplyDiscount(int price, int discount, int expected) {
    assertEquals(expected, applyDiscount(price, discount));
}
```

I use parameterized tests aggressively. If I'm writing the same test structure more than twice with different inputs, it becomes a parameterized test.

## Dynamic Tests: For When Parameterized Isn't Enough

Dynamic tests are generated at runtime. Useful when your test cases come from external data or complex computation:

```java
@TestFactory
Stream<DynamicTest> testAllConfigFiles() {
    return Files.list(Path.of("src/test/resources/configs"))
        .map(path -> dynamicTest(
            "Config " + path.getFileName(),
            () -> assertDoesNotThrow(() -> ConfigParser.parse(path))
        ));
}
```

The test report shows each dynamic test individually. I use this for testing schema migrations, configuration files, and anything where the test cases are data-driven.

## Extensions: Replacing @Rule

JUnit 4 had `@Rule` and `@ClassRule`. JUnit 5 has extensions, which are more powerful and composable.

```java
public class TimingExtension implements BeforeTestExecutionCallback, AfterTestExecutionCallback {
    private static final Logger log = LoggerFactory.getLogger(TimingExtension.class);

    @Override
    public void beforeTestExecution(ExtensionContext ctx) {
        getStore(ctx).put("start", System.currentTimeMillis());
    }

    @Override
    public void afterTestExecution(ExtensionContext ctx) {
        long start = getStore(ctx).get("start", long.class);
        long duration = System.currentTimeMillis() - start;
        log.info("{} took {}ms", ctx.getDisplayName(), duration);
    }

    private ExtensionContext.Store getStore(ExtensionContext ctx) {
        return ctx.getStore(ExtensionContext.Namespace.create(getClass(), ctx.getRequiredTestMethod()));
    }
}

@ExtendWith(TimingExtension.class)
class SlowTest { ... }
```

The extension points are: `BeforeAllCallback`, `BeforeEachCallback`, `BeforeTestExecutionCallback`, `AfterTestExecutionCallback`, `AfterEachCallback`, `AfterAllCallback`, `ParameterResolver`, `TestInstancePostProcessor`, and more.

Spring's `@SpringBootTest` is just a JUnit 5 extension (`SpringExtension`). Testcontainers is just a JUnit 5 extension. Once you understand the extension model, you understand how all the test integrations work.

## Assertions: JUnit vs AssertJ

JUnit 5 ships with basic assertions:

```java
assertEquals(expected, actual);
assertTrue(condition);
assertThrows(Exception.class, () -> code());
assertAll(
    () -> assertEquals("John", user.name()),
    () -> assertEquals("john@example.com", user.email())
);
```

`assertAll` is useful - it runs all assertions and reports all failures, instead of stopping at the first one. Good for validating DTOs.

But honestly? I use AssertJ for almost everything:

```java
assertThat(users)
    .hasSize(3)
    .extracting(User::name)
    .containsExactly("Alice", "Bob", "Charlie");

assertThat(result)
    .isInstanceOf(Success.class)
    .extracting("transactionId")
    .isNotNull();

assertThatThrownBy(() -> service.process(null))
    .isInstanceOf(IllegalArgumentException.class)
    .hasMessageContaining("must not be null");
```

AssertJ's fluent API is more readable, has better error messages, and supports complex collection assertions out of the box. The migration from JUnit assertions to AssertJ is one of the highest-ROI changes you can make to a test suite.

## Nested Tests: Organize by Scenario

```java
class OrderServiceTest {
    @Nested
    @DisplayName("When creating an order")
    class WhenCreating {
        @Test
        void shouldRejectEmptyItems() { ... }

        @Test
        void shouldCalculateTotal() { ... }
    }

    @Nested
    @DisplayName("When cancelling an order")
    class WhenCancelling {
        @Test
        void shouldRequireReason() { ... }

        @Test
        void shouldRefundPayment() { ... }
    }
}
```

The test report becomes a readable hierarchy. I use nested classes to group tests by scenario, state, or feature. It makes large test classes navigable.

## Tags and Filtering

```java
@Tag("slow")
@Tag("integration")
@Test
void shouldSyncWithExternalSystem() { ... }
```

In Maven:

```xml
<plugin>
    <artifactId>maven-surefire-plugin</artifactId>
    <configuration>
        <excludedGroups>slow</excludedGroups>
    </configuration>
</plugin>
```

I tag tests as `unit`, `integration`, `slow`, or `flaky`. The CI pipeline runs unit tests on every commit, integration tests on every PR, and the full suite nightly. Flaky tests get their own quarantine run where failures don't block the build. Yes, I know quarantining flaky tests is a crutch. But it's better than ignoring them.

## The Pattern I Use Everywhere

Every test class in my projects follows this structure:

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock OrderRepository orderRepository;
    @Mock PaymentGateway paymentGateway;
    @InjectMocks OrderService orderService;

    @Test
    @DisplayName("Should create order and process payment")
    void shouldCreateOrderAndProcessPayment() {
        // given
        var items = List.of(new LineItem("widget", 2, TEN));
        when(paymentGateway.charge(any())).thenReturn(new PaymentResult.Success("tx-123"));

        // when
        var result = orderService.create("customer-1", items);

        // then
        assertThat(result.status()).isEqualTo(OrderStatus.CONFIRMED);
        verify(orderRepository).save(any(Order.class));
    }
}
```

Given/when/then. Mockito for dependencies. AssertJ for assertions. Display names for humans. It's not clever, but it's consistent and readable six months later.
